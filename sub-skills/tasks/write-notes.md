---
name: autopku-task-write-notes
description: Standard AutoPku notes pipeline: one PDF per writer agent, PDF text extraction via pdf-reader, per-lecture Markdown notes, README index, and optional polished PDF export.
---

# 任务：撰写课程笔记

本任务是 AutoPku 的标准笔记 pipeline。核心原则：

- 每个课件 PDF 独立分配一个 Writer Agent 并行处理。
- Writer Agent 必须按 `sub-skills/tools/pdf-reader.md` 的 PyMuPDF/pdfplumber 方案读取 PDF。
- 每个 PDF 只产出一个单篇笔记：`notes/{lecture_name}.md`。
- Coordinator 等全部 Writer Agent 完成后，再生成 `notes/README.md` 作为课程笔记索引。
- 如用户要求合成 PDF，再用 Pandoc + Typst/LaTeX 风格模板从 `notes/` 生成排版文档。

## 触发场景

用户表达以下意图时使用本任务：

- "给某门课写笔记"
- "整理课件"
- "把 PDF 课件整理成 notes"
- "生成课程笔记 PDF"

## 前置确认

如果用户没有明确给出范围，询问最少必要问题：

```python
AskUserQuestion({
    "questions": [
        {
            "question": "选择要撰写笔记的课件：",
            "options": [
                {"label": "全部课件", "value": "all"},
                {"label": "指定范围", "value": "range"}
            ],
            "multiSelect": False
        },
        {
            "question": "笔记详细程度：",
            "options": [
                {"label": "精简（只保留核心定义、结论和考试点）", "value": "minimal"},
                {"label": "标准（包含证明思路、关键机制和例子）", "value": "standard"},
                {"label": "详细（完整推导、代码/实验步骤和扩展说明）", "value": "detailed"}
            ],
            "multiSelect": False
        },
        {
            "question": "是否需要最终合成 PDF 文档：",
            "options": [
                {"label": "需要", "value": "pdf"},
                {"label": "暂不需要", "value": "markdown_only"}
            ],
            "multiSelect": False
        }
    ]
})
```

如果用户已经明确课程、PDF 目录或输出形式，直接执行，不重复询问。

## 标准目录约定

```text
{course_dir}/
├── lectures/                 # 推荐课件目录；也可使用用户指定 PDF 目录
│   ├── 01-topic.pdf
│   └── 02-topic.pdf
├── notes/
│   ├── 01-topic.md
│   ├── 02-topic.md
│   └── README.md
└── pdf/                      # 仅在用户要求合成文档时创建
    ├── {course_name}课程笔记.typ
    ├── {course_name}课程笔记.pdf
    └── notes-template.typ
```

## Coordinator Pipeline

Coordinator 负责发现输入、创建 Writer Agent、收敛结果、生成索引和验证输出。

1. **发现课件**
   - 优先使用用户指定目录。
   - 未指定时，在课程目录中依次查找 `lectures/`、`课件/`、`slides/`、`materials/`。
   - 只处理 `.pdf` 文件，按文件名自然排序。

2. **准备输出目录**
   - 创建 `notes/`。
   - 为每个 PDF 生成稳定文件名：保留课程顺序前缀，去掉不可用于文件名的字符，输出为 `notes/{lecture_name}.md`。
   - 如目标文件已存在，除非用户要求覆盖，否则先读取并判断是否需要增量更新。

3. **并行 Writer Agents**
   - 为每个 PDF 创建一个 Writer Agent。
   - 每个 Writer Agent 只拥有自己的 `{pdf_path}` 和 `{note_path}` 写入范围。
   - Writer Agent 之间不能共享同一个输出文件，不能修改 `notes/README.md`。
   - Codex 环境使用 native subagents；Claude/Kimi 环境使用各自 Agent Team；不支持并行时串行执行同一 Writer prompt。

4. **生成索引**
   - 等全部 Writer Agent 完成后，Coordinator 读取每个 note 的标题、摘要、关键概念。
   - 生成或更新 `notes/README.md`，作为课程级入口。

5. **可选 PDF 合成**
   - 用户要求 PDF 时，使用 `notes/README.md` + 所有单篇 note 作为输入。
   - 首选 Pandoc + Typst 生成轻量、好看的 LaTeX 风格 PDF；若环境已配置 XeLaTeX，也可使用 LaTeX 模板。
   - 输出到 `pdf/{course_name}课程笔记.pdf`，不要覆盖原始课件 PDF。

6. **验证**
   - 检查每个 PDF 都有对应 `notes/{lecture_name}.md`。
   - 检查 `notes/README.md` 链接的文件都存在。
   - 如生成 PDF，检查 PDF 存在、页数大于 0，并抽取首页文本确认不是空白文档。

## Writer Agent Prompt

每个 PDF 使用以下模板创建独立 Writer Agent：

````text
你是 AutoPku 笔记 Writer Agent，负责把一个课程 PDF 整理成一篇可复习 Markdown 笔记。

输入 PDF: {pdf_path}
输出笔记: {note_path}
详细程度: {detail_level}

写入范围:
- 只能创建或更新 {note_path}
- 不能修改其他笔记
- 不能修改 notes/README.md

必须引用工具:
- 阅读 sub-skills/tools/pdf-reader.md
- 优先使用 PyMuPDF 读取全文
- 遇到表格或 PyMuPDF 抽取质量差时，用 pdfplumber 补读对应页

推荐读取代码:
```python
from pathlib import Path
import fitz

pdf_path = Path("{pdf_path}")
with fitz.open(pdf_path) as doc:
    pages = [(i + 1, page.get_text("text")) for i, page in enumerate(doc)]
```

内容筛选:
- 保留课程核心问题、定义、原理、机制、定理、算法、实验步骤、关键代码/命令、易错点。
- 去除重复页眉页脚、课堂闲聊、纯装饰性图片说明、无助于复习的背景故事。
- 对公式使用 LaTeX：行内 `$...$`，行间 `$$...$$`。
- 对代码和命令使用 fenced code block。
- 对无法可靠抽取的图表，标注页码并用一句话说明需要人工查看。

输出格式:
```markdown
# {lecture_title}

> 来源：{pdf_filename}
> 页数：{page_count}

## 本讲主线
- ...

## 核心概念

### {concept}
...

## 关键机制 / 推导 / 算法
...

## 例题 / 实验 / 命令
...

## 易错点
- ...

## 速查
| 项 | 内容 |
|----|------|
| ... | ... |

## 仍需人工确认
- 第 X 页：...
```

完成报告:
- PDF 页数
- 输出文件路径
- 主要章节数
- 无法抽取或需要人工确认的页码
````

## README 索引格式

Coordinator 生成 `notes/README.md`：

```markdown
# {course_name} 笔记索引

生成时间: {timestamp}
课件数量: {pdf_count}
笔记数量: {note_count}

## 阅读顺序

| 序号 | 课件 | 笔记 | 核心内容 |
|-----|------|------|----------|
| 1 | 01-topic.pdf | [01-topic.md](01-topic.md) | ... |

## 课程主线
- ...

## 关键概念地图
- A -> B -> C

## 复习速查
- ...

## 生成说明
- 每个 PDF 由独立 Writer Agent 处理。
- PDF 文本读取采用 `sub-skills/tools/pdf-reader.md` 中的 PyMuPDF/pdfplumber 方案。
```

## PDF 合成规范

当用户要求"组成一个 PDF 文档"或类似意图时执行：

1. 确认可用工具：`pandoc`、`typst`；如果用户明确要求 LaTeX，再检查 `xelatex` 或 `lualatex`。
2. 在 `pdf/` 下创建排版模板，例如 `notes-template.typ` 或 `notes-template.tex`。
3. 使用 `notes/README.md` 和所有 `notes/*.md` 生成单一源文件。
4. 编译输出 `pdf/{course_name}课程笔记.pdf`。
5. 验证 PDF 页数和首页文本，避免生成空白 PDF。

示例命令：

```bash
pandoc notes/README.md notes/*.md \
  --from markdown+smart \
  --to typst \
  --standalone \
  --toc --toc-depth=2 \
  --metadata title="{course_name}课程笔记" \
  -V template="notes-template.typ" \
  -o "pdf/{course_name}课程笔记.typ"

typst compile "pdf/{course_name}课程笔记.typ" "pdf/{course_name}课程笔记.pdf"
```

## 使用示例

```bash
skill: autopku notes 逻辑导论
skill: autopku notes /path/to/course/lectures /path/to/course/notes
skill: autopku notes 操作系统 --pdf
```
