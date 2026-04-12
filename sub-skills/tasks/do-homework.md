---
name: autopku-task-do-homework
description: 完成PKU课程作业（解析→解答→渲染→询问用户→提交）
---

# 任务：完成作业并提交

## 完整流程

```
Phase 1: PDF解析 → Phase 2: 解答 → Phase 3: 渲染 → 
询问用户（展示完成的作业）→ 
[用户确认] → Phase 4: 提交到教学网
```

## 前置步骤：用户确认

### 1. 列出该课程所有待交作业

```bash
/tmp/pku3b a ls --all-term | grep -i "{course_name}"
```

### 2. 使用 AskUserQuestion 询问用户

```python
AskUserQuestion({
    "questions": [{
        "question": "请选择要完成的作业：",
        "options": [
            {"label": "第一次习题 (截止: 3天后)", "value": "hw1"},
            {"label": "第二次习题 (截止: 5天后)", "value": "hw2"}
        ],
        "multiSelect": False
    }]
})
```

### 3. 确认本地PDF存在

```bash
ls -la "{base_dir}/{course}/作业/Homework*.pdf"
```

### 4. 二次确认

```python
AskUserQuestion({
    "questions": [{
        "question": "确认要为 {course} 完成 {assignment} 吗？",
        "options": [
            {"label": "确认开始", "value": "confirm"},
            {"label": "取消", "value": "cancel"}
        ]
    }]
})
```

## Phase 1: PDF 解析

引用: `sub-skills/tools/pdf-reader.md`

```python
import pdfplumber
import json
import re

def parse_homework(pdf_path, output_json):
    content = {'pages': [], 'problems': []}
    
    with pdfplumber.open(pdf_path) as pdf:
        for i, page in enumerate(pdf.pages):
            text = page.extract_text()
            content['pages'].append({'page_num': i+1, 'text': text})
    
    full_text = '\n'.join(p['text'] for p in content['pages'])
    
    # 提取题目
    pattern = r'(?:^|\n)\s*(?:Problem\s*)?(\d+)[\.、\)]\s*([^\n]+)(.*?)(?=\n(?:\d+|Problem|\Z))'
    matches = re.findall(pattern, full_text, re.DOTALL | re.IGNORECASE)
    
    for num, title, body in matches:
        content['problems'].append({
            'number': num.strip(),
            'title': title.strip(),
            'content': body.strip()
        })
    
    with open(output_json, 'w', encoding='utf-8') as f:
        json.dump(content, f, ensure_ascii=False, indent=2)
    
    return content

# 执行
content = parse_homework(
    f"{base_dir}/{course}/作业/{pdf_file}",
    f"{base_dir}/{course}/作业/homework_parsed.json"
)
print(f"解析完成，找到 {len(content['problems'])} 道题目")
```

## Phase 2: 解答

引用: `sub-skills/runtime/create-agent.md`
引用: `sub-skills/tools/agent-helpers.md` (solver模板)

```
你是 Solver，完成 {course} {assignment} 的解答。

输入：{base_dir}/{course}/作业/homework_parsed.json
资料：{base_dir}/{course}/资料/
输出：{base_dir}/{course}/作业/answers.json

任务：
1. 读取解析的题目
2. 检查资料目录，读取相关文件
3. 逐题解答（公式、推导、答案）
4. 标注参考资料
5. 保存 JSON：{"answers": [{"problem_number": "1", "solution_steps": [], "final_answer": ""}]}
```

## Phase 3: 渲染

引用: `sub-skills/tools/agent-helpers.md` (writer模板)

### 生成 Markdown

```
你是 Writer，生成提交文档。

输入：{answers_json}
输出：{base_dir}/{course}/作业/{assignment}_answer.md

格式：
# {course} - {assignment} 答案
**姓名：** ____  **学号：** ____  **日期：** {date}

## 第 1 题
**题目：** ...
**解答：** ...
**答案：** ...
```

### Markdown → PDF

```bash
pip3 install markdown

python3 << 'PYEOF'
import markdown
import sys
sys.path.insert(0, '/Users/moonshot/Library/Python/3.9/lib/python/site-packages')

with open('{md_path}', 'r', encoding='utf-8') as f:
    md = f.read()

html = markdown.markdown(md, extensions=['tables', 'fenced_code'])
html_full = f'''<!DOCTYPE html><html><head><meta charset="utf-8">
<style>body {{ font-family: "Noto Serif CJK SC", serif; margin: 40px; line-height: 1.8; }}
h1 {{ text-align: center; border-bottom: 2px solid #333; }}</style>
<script src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
</head><body>{html}</body></html>'''

with open('{html_path}', 'w', encoding='utf-8') as f:
    f.write(html_full)
PYEOF

"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless --disable-gpu --print-to-pdf="{pdf_path}" \
    --run-all-compositor-stages-before-draw "file://{html_path}"
```

## 【询问用户】

```python
AskUserQuestion({
    "questions": [{
        "question": "作业已完成渲染。是否提交到教学网？",
        "options": [
            {"label": "查看PDF后提交", "value": "submit"},
            {"label": "先不提交，保存到本地", "value": "save_only"},
            {"label": "取消", "value": "cancel"}
        ]
    }]
})
```

## Phase 4: 提交（用户确认后）

引用: `sub-skills/tools/pku3b-setup.md`

```bash
# 1. 查找作业ID
/tmp/pku3b a ls --all-term | grep -i "{course}"

# 2. 提交
/tmp/pku3b a submit {assignment_id} "{pdf_path}"

# 3. 验证
/tmp/pku3b a ls --all-term | grep -i "{assignment}"
```

## Agent Team 结构（可选）

引用: `sub-skills/runtime/create-agent.md`

```
Coordinator
├── Parser (Phase 1)
├── Solver (Phase 2)
├── Writer (Phase 3)
└── Submitter (Phase 4, 用户确认后)
```

## 输出

- `{course}/作业/{assignment}_answer.md`
- `{course}/作业/{assignment}_answer.pdf`
- `{course}/提交/{assignment}_answer.pdf`（保存时）
- 教学网提交状态
