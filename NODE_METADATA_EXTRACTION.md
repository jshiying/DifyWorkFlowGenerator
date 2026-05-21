# Dify 节点元数据自动抽取方案

自动从 Dify 源码提取各工作流节点的参数定义，生成与 `workflow_generator_prompt_cn.yml` 格式一致的提示词文件。

---

## 整体流水线

```
GitHub API 拉取所有 entities.py
         ↓
Python AST 解析 Pydantic 模型字段
         ↓
生成结构化 JSON Schema
         ↓
调用 LLM API 生成中文 YAML 片段
         ↓
自动组装 workflow_generator_prompt.yml
```

---

## 背景知识

### 节点源码位置

Dify 开源仓库中，每个工作流节点的参数定义遵循统一结构：

```
api/core/workflow/nodes/
├── base/
│   └── entities.py        # BaseNodeData（所有节点公共字段）
├── llm/
│   └── entities.py        # LLMNodeData
├── start/
│   └── entities.py        # StartNodeData
├── end/
│   └── entities.py        # EndNodeData
├── if_else/
│   └── entities.py        # IfElseNodeData
├── question_classifier/
│   └── entities.py
├── http_request/
│   └── entities.py
├── code/
│   └── entities.py
├── knowledge_retrieval/
│   └── entities.py
├── variable_aggregator/
│   └── entities.py
├── template_transform/
│   └── entities.py
├── parameter_extractor/
│   └── entities.py
├── answer/
│   └── entities.py
└── tool/
    └── entities.py        # 通用工具节点
```

工具节点（Jina、Tavily、YouTube 等）的参数定义在：

```
api/core/tools/provider/builtin/{工具名}/tools/{工具名}.yaml
```

### Raw 文件 URL 规律

```
https://raw.githubusercontent.com/langgenius/dify/{版本}/api/core/workflow/nodes/{节点名}/entities.py
```

例如 Dify 1.0.0 的 LLM 节点：

```
https://raw.githubusercontent.com/langgenius/dify/1.0.0/api/core/workflow/nodes/llm/entities.py
```

### prompt 文件的三层结构

| 层 | 字段 | 来源 |
|---|---|---|
| 字段骨架 | `required_structure` | entities.py（Pydantic 模型） |
| 使用说明 | `usage_guide` | entities.py + 人工经验 |
| 完整示例 | `output_format` | 实际导出的 DSL YAML |

---

## 各脚本说明

| 脚本 | 输入 | 输出 | 是否需要 LLM |
|------|------|------|-------------|
| `fetch_entities.py` | GitHub Raw URL | `*.py` 源码文件 | 否 |
| `parse_entities.py` | `*.py` 源码 | `all_schemas.json` | 否 |
| `generate_prompt.py` | JSON Schema + 源码 | `workflow_generator_prompt_auto.yml` | 是（一次性） |

---

## 脚本一：自动拉取所有节点的 entities.py

```python
# fetch_entities.py
"""
自动从 GitHub 拉取指定 Dify 版本下所有节点的 entities.py
"""
import requests
import json
from pathlib import Path

DIFY_VERSION = "1.0.0"
NODES_API_URL = f"https://api.github.com/repos/langgenius/dify/contents/api/core/workflow/nodes?ref={DIFY_VERSION}"
RAW_BASE = f"https://raw.githubusercontent.com/langgenius/dify/{DIFY_VERSION}"
OUTPUT_DIR = Path(f"dify_nodes_{DIFY_VERSION}")


def fetch_all_entities():
    OUTPUT_DIR.mkdir(exist_ok=True)

    # 1. 获取节点目录列表
    resp = requests.get(NODES_API_URL)
    items = resp.json()
    node_dirs = [item["name"] for item in items if item["type"] == "dir"]
    print(f"发现节点目录：{node_dirs}")

    results = {}

    # 2. 逐个拉取 entities.py
    for node_name in node_dirs:
        url = f"{RAW_BASE}/api/core/workflow/nodes/{node_name}/entities.py"
        r = requests.get(url)
        if r.status_code == 200:
            content = r.text
            (OUTPUT_DIR / f"{node_name}.py").write_text(content, encoding="utf-8")
            results[node_name] = content
            print(f"✅ {node_name}/entities.py")
        else:
            print(f"⚠️  {node_name}/entities.py 不存在，跳过")

    # 3. 额外拉取内置工具节点的 YAML 定义
    tool_nodes = {
        "jina_reader": "api/core/tools/provider/builtin/jina/tools/jina_reader.yaml",
        "tavily_search": "api/core/tools/provider/builtin/tavily/tools/tavily_search.yaml",
        "youtube_transcript": "api/core/tools/provider/builtin/transcript/tools/free_youtube_transcript.yaml",
    }
    for tool_name, path in tool_nodes.items():
        url = f"{RAW_BASE}/{path}"
        r = requests.get(url)
        if r.status_code == 200:
            (OUTPUT_DIR / f"tool_{tool_name}.yaml").write_text(r.text, encoding="utf-8")
            results[f"tool_{tool_name}"] = r.text
            print(f"✅ 工具节点 {tool_name}")

    # 保存清单
    (OUTPUT_DIR / "manifest.json").write_text(
        json.dumps(list(results.keys()), ensure_ascii=False, indent=2),
        encoding="utf-8",
    )
    return results


if __name__ == "__main__":
    fetch_all_entities()
```

---

## 脚本二：用 AST 解析 Pydantic 模型，生成结构化 JSON

```python
# parse_entities.py
"""
用 Python AST 解析 entities.py，提取 Pydantic 模型字段定义。
纯静态分析，无需安装 Dify 依赖。
"""
import ast
import json
from pathlib import Path

DIFY_VERSION = "1.0.0"
INPUT_DIR = Path(f"dify_nodes_{DIFY_VERSION}")


def parse_pydantic_class(source_code: str) -> dict:
    """从源码中提取所有 Pydantic BaseModel 子类及其字段"""
    try:
        tree = ast.parse(source_code)
    except SyntaxError:
        return {}

    classes = {}

    for node in ast.walk(tree):
        if not isinstance(node, ast.ClassDef):
            continue

        bases = [
            (b.id if isinstance(b, ast.Name) else
             b.attr if isinstance(b, ast.Attribute) else "")
            for b in node.bases
        ]
        if not any("Model" in b or "Data" in b or "Config" in b for b in bases):
            continue

        fields = {}
        for item in node.body:
            if not isinstance(item, ast.AnnAssign):
                continue
            if not isinstance(item.target, ast.Name):
                continue

            field_name = item.target.id
            field_type = ast.unparse(item.annotation)

            default = None
            required = True
            if item.value is not None:
                default_src = ast.unparse(item.value)
                required = False
                if "Field(...)" in default_src or default_src == "...":
                    required = True
                    default = None
                else:
                    default = default_src

            is_optional = (
                "Optional" in field_type
                or "None" in field_type
                or not required
            )

            fields[field_name] = {
                "type": field_type,
                "required": required and not is_optional,
                "optional": is_optional,
                "default": default,
            }

        if fields:
            classes[node.name] = {
                "bases": bases,
                "fields": fields,
            }

    return classes


def process_all_nodes():
    all_schemas = {}

    for py_file in INPUT_DIR.glob("*.py"):
        node_name = py_file.stem
        source = py_file.read_text(encoding="utf-8")
        classes = parse_pydantic_class(source)
        if classes:
            all_schemas[node_name] = classes
            print(f"✅ 解析 {node_name}: {list(classes.keys())}")

    output_path = INPUT_DIR / "all_schemas.json"
    output_path.write_text(
        json.dumps(all_schemas, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )
    print(f"\n📄 已输出 Schema: {output_path}")
    return all_schemas


if __name__ == "__main__":
    process_all_nodes()
```

---

## 脚本三：调用 LLM API 生成 YAML 片段，自动组装最终文件

```python
# generate_prompt.py
"""
调用 OpenAI API，将 JSON Schema 转换为 workflow_generator_prompt.yml
"""
import json
import time
from pathlib import Path
from openai import OpenAI

DIFY_VERSION = "1.0.0"
INPUT_DIR = Path(f"dify_nodes_{DIFY_VERSION}")
OUTPUT_FILE = Path("workflow_generator_prompt_auto.yml")

client = OpenAI()  # 读取环境变量 OPENAI_API_KEY

# 节点处理顺序（按重要性）
NODE_ORDER = [
    "start", "llm", "end", "if_else", "question_classifier",
    "http_request", "code", "knowledge_retrieval",
    "variable_aggregator", "template_transform",
    "parameter_extractor", "answer", "iteration",
]

SYSTEM_PROMPT = """你是 Dify 工作流 DSL 专家，熟悉 Dify 工作流 YAML 格式规范。
你的任务是根据 Pydantic 模型的 JSON Schema，生成用于指导 LLM 生成 Dify 工作流的中文 YAML 提示词片段。
输出必须是合法 YAML，使用中文注释和说明。"""


def generate_node_section(node_name: str, schema: dict, source_code: str) -> str:
    """调用 LLM 为单个节点生成 required_structure + usage_guide 片段"""

    user_prompt = f"""
节点名称：{node_name}
Dify 版本：{DIFY_VERSION}

Pydantic Schema（JSON 格式）：
{json.dumps(schema, ensure_ascii=False, indent=2)}

原始源码参考：
```python
{source_code[:3000]}
```

请生成两个 YAML 片段：

【片段1】required_structure 部分（缩进4空格，作为 nodes 下的子键）：
{node_name}_node:
  - id: 唯一ID
  - type: [DSL类型名]（固定）
  - data:
    # 必填字段在前，可选字段在后
    # Optional 字段注明"（可选）"，有默认值注明"默认=xxx"
    # Literal 类型列出所有合法值

【片段2】usage_guide 部分（纯文本，缩进6空格）：
N. {node_name}节点
   使用场景:
   - [场景]
   结构说明:
   - [关键字段]
   注意事项:
   - [约束和陷阱]

注意：
- DSL 中节点 type 字段用连字符（if_else → if-else，http_request → http-request）
- 嵌套 BaseModel 展开到叶子字段
- 工作流中变量引用格式：{{#节点ID.变量名#}}
- 只输出 YAML，不要解释
"""

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_prompt},
        ],
        temperature=0.1,
    )
    return response.choices[0].message.content


def assemble_prompt_file(node_sections: dict) -> str:
    """将所有节点片段组装成完整的 prompt YAML 文件"""

    required_parts = []
    usage_parts = []

    for node_name in NODE_ORDER:
        if node_name not in node_sections:
            continue
        section = node_sections[node_name]
        if "【片段2】" in section:
            parts = section.split("【片段2】")
            required_parts.append(parts[0].replace("【片段1】", "").strip())
            usage_parts.append(parts[1].strip())
        else:
            required_parts.append(section.strip())

    final_yml = f"""prompt:
  metadata:
    title: Dify工作流生成提示词
    version: {DIFY_VERSION}
    description: 用于生成Dify工作流文件的提示词定义

  input_requirements: |
    请提供以下信息：
    1. 工作流的目的
    2. 作为输入所需的信息
    3. 作为输出所期望的格式和内容
    4. 特殊约束条件

  instructions:
    workflow_overview: |
      根据所提供的需求，生成包含以下要素的Dify工作流文件（YAML格式）：
      - 由节点构成的工作流
      - 节点间按顺序执行连接
      - 自动化从输入到输出的一系列处理

    required_structure:
      app_info:
        - mode: workflow
        - name: 体现目的的名称
        - version: 0.1.3

      workflow_graph:
        nodes:
{chr(10).join(required_parts)}

    usage_guide: |
      通用事项：
      - 节点ID必须唯一，范围 1700000000000-1799999999999

{chr(10).join(usage_parts)}

    output_format: |
      [完整 YAML 示例由实际 DSL 导出补充]
"""
    return final_yml


def main():
    schema_path = INPUT_DIR / "all_schemas.json"
    all_schemas = json.loads(schema_path.read_text(encoding="utf-8"))

    node_sections = {}

    for node_name in NODE_ORDER:
        if node_name not in all_schemas:
            print(f"⚠️  {node_name} 无 Schema，跳过")
            continue

        source_file = INPUT_DIR / f"{node_name}.py"
        source_code = source_file.read_text(encoding="utf-8") if source_file.exists() else ""

        print(f"🤖 正在生成 {node_name} 节点片段...")
        try:
            section = generate_node_section(node_name, all_schemas[node_name], source_code)
            node_sections[node_name] = section

            # 缓存单节点结果，防止中途失败丢失
            cache_file = INPUT_DIR / f"generated_{node_name}.yml"
            cache_file.write_text(section, encoding="utf-8")

            time.sleep(1)  # 避免触发速率限制
        except Exception as e:
            print(f"❌ {node_name} 生成失败: {e}")

    final_content = assemble_prompt_file(node_sections)
    OUTPUT_FILE.write_text(final_content, encoding="utf-8")
    print(f"\n✅ 已生成：{OUTPUT_FILE}")


if __name__ == "__main__":
    main()
```

---

## 一键运行

```bash
# 安装依赖
pip install requests openai pyyaml

# 设置 OpenAI API Key
export OPENAI_API_KEY=your_key_here   # Linux/macOS
set OPENAI_API_KEY=your_key_here      # Windows

# 按顺序执行三个脚本
python fetch_entities.py     # 拉取所有节点源码
python parse_entities.py     # 解析 Pydantic 字段
python generate_prompt.py    # LLM 生成并组装
```

---

## 版本更新

切换 Dify 版本时，只需修改 `DIFY_VERSION` 变量后重跑：

```bash
# 例：升级到 1.13.2
DIFY_VERSION=1.13.2 python fetch_entities.py && \
DIFY_VERSION=1.13.2 python parse_entities.py && \
DIFY_VERSION=1.13.2 python generate_prompt.py
```

也可以使用 GitHub Compare 快速定位变更节点：

```
https://github.com/langgenius/dify/compare/1.0.0...1.13.2
```

在文件列表中筛选 `api/core/workflow/nodes/` 路径，只对有变化的节点重新生成对应片段。

---

## 注意事项

### entities.py 无法覆盖的内容

以下内容需从实际 DSL 导出补充，无法从源码自动抽取：

| 内容 | 原因 | 获取方式 |
|------|------|---------|
| `output_format` 完整示例 | 源码定义合法范围，不含具体值 | 在 Dify UI 配置节点后导出 DSL |
| 字段在 DSL 中的连字符写法 | 源码用下划线，DSL 用连字符 | 导出 DSL 对比 |
| 工具节点参数（Jina/Tavily） | 不在 `nodes/` 下 | 读 `tools/provider/builtin/` |
| `edges` 结构 | 运行时动态生成 | 导出 DSL 观察 |

### 字段名转换规则

| 源码（Python） | DSL（YAML） |
|---------------|------------|
| `if_else` | `if-else` |
| `http_request` | `http-request` |
| `knowledge_retrieval` | `knowledge-retrieval` |
| `question_classifier` | `question-classifier` |
| `variable_aggregator` | `variable-aggregator` |
| `template_transform` | `template-transform` |
| `parameter_extractor` | `parameter-extractor` |
