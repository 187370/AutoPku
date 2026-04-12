---
name: autopku-task-sync-notices
description: 同步PKU课程通知，为每门课程创建目录结构和摘要
---

# 任务：同步课程通知

## 执行流程

1. **环境准备**
   - 引用: `sub-skills/tools/pku3b-setup.md`
   - 检查/安装 pku3b，登录教学网

2. **获取数据**
   - 引用: `sub-skills/tools/data-parser.md`
   - 获取作业列表、公告、选课信息
   - 解析并筛选当前学期课程

3. **并行处理**
   - 引用: `sub-skills/runtime/create-agent.md`
   - 为每门课程创建 agent
   - 引用: `sub-skills/tools/agent-helpers.md` (通知摘要模板)

4. **生成汇总**
   - 收集所有 agent 结果
   - 生成通知摘要汇总.md

## 详细步骤

### 步骤1: 检查并安装 pku3b

```bash
which pku3b 2>/dev/null || echo "NOT_FOUND"
```

如未安装，按 `pku3b-setup.md` 安装。

### 步骤2: 登录

使用 expect 脚本登录（TTY-safe）。

### 步骤3: 获取作业数据

```bash
/tmp/pku3b a ls --all-term > /tmp/pku_assignments_raw.txt
/tmp/pku3b ann ls > /tmp/pku_announcements_raw.txt
/tmp/pku3b s -d major show > /tmp/major_courses.txt 2>/dev/null || true
```

### 步骤4: 解析数据

```python
import re
import json

# 解析作业
with open('/tmp/pku_assignments_raw.txt') as f:
    raw = f.read()
pattern = r'\x1b\[36m\x1b\[1m([^\x1b]+?)\x1b\[0m\x1b\[0m\s+\x1b\[2m>\x1b\[0m\s+\x1b\[36m\x1b\[1m([^\x1b]+?)\x1b\[0m\x1b\[0m\s+\(([^)]+)\)'
assignments = [{'course': c.strip(), 'assignment': a.strip(), 'status': s.strip()}
               for c, a, s in re.findall(pattern, raw)]

with open('/tmp/pku_assignments.json', 'w') as f:
    json.dump(assignments, f, ensure_ascii=False, indent=2)

# 解析课程（如有）
try:
    with open('/tmp/major_courses.txt') as f:
        courses = re.findall(r'已选上.*?\x1b\[32m([^\x1b]+)\x1b\[0m', f.read())
except:
    courses = list(set(a['course'] for a in assignments))
```

### 步骤5: 并行创建 Agents

引用: `sub-skills/runtime/create-agent.md`

为每个课程创建 agent：

```
你是课程 "{course}" 的专属 agent。

工作目录：{base_dir}/{course}/

任务：
1. 创建目录：作业/、通知/、资料/
2. 筛选 /tmp/pku_assignments.json 中 course="{course}" 的作业
3. 下载附件（如有）：/tmp/pku3b a download <ID> -d {course}/作业/
4. 生成 {course}/通知摘要.md：
   - 统计：总作业数、待交、已完成、逾期
   - 待完成作业（🔴紧急<1天 🟡临期<7天 🟢正常）
   - 逾期列表、已完成列表
   - 下载文件列表

返回：找到X个作业，待交Y个，下载Z个文件
```

### 步骤6: 生成汇总报告

```python
from datetime import datetime

report = f"""# PKU 课程通知汇总
生成时间：{datetime.now():%Y-%m-%d %H:%M}

## 统计
- 课程数：{len(courses)}
- 总作业：{len(assignments)}
- 待交：{sum(1 for a in assignments if '已完成' not in a['status'])}

## 课程清单
{course_list}

*由 AutoPku 自动生成*
"""

with open(f'{base_dir}/通知摘要汇总.md', 'w') as f:
    f.write(report)
```

## 输出

- `test/通知摘要汇总.md`
- `test/{course}/通知摘要.md`（每个课程）
- `test/{course}/作业/`（下载的附件）
