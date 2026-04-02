# AutoPku

> 北京大学本科全自动解决方案 | 从通知获取到作业提交，全程 AI 托管

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Agent Team](https://img.shields.io/badge/Agent%20Team-Experimental-blue)](https://github.com/anthropics/claude-code)
[![pku3b](https://img.shields.io/badge/pku3b-Compatible-green)](https://github.com/sshwy/pku3b)

**只需学号密码 → AI 自动完成一切**

![AutoPku 流程概览](./images/autopku-flow.png)

---
## 快速开始

### 第一步：下载 Skill

```bash
git clone https://github.com/ICUlizhi/AutoPku.git
```

### 第二步：交给 Claude Code

在 Claude Code 中加载 skill：

```
读取 AutoPku/skill.md，并执行
```

然后 Claude 会让你告诉它学号和密码：

```
学号是 2200011111，密码是 xxx
```

### 第三步：离开电脑，相信cc
去享受北平难得的春天

---
## 项目背景

当前 AI Agent 领域存在一个根本性矛盾：我们一方面期待 Agent 能够处理复杂的真实世界任务，另一方面现有的单 Agent 架构却在上下文管理、任务并行和长期规划上表现出明显的瓶颈。以 AutoGPT、OpenClaw 等为代表的早期探索虽然验证了概念可行性，但在面对多课程并行、长链路依赖的场景时，串行执行导致的效率低下和上下文遗忘问题变得难以忽视。

Agent Team 架构提供了一种不同的思路。与其在单个 Agent 的上下文窗口内不断堆叠复杂度，不如将任务天然地拆分为多个独立执行的单元，每个单元由专门的 Agent 负责，通过结构化的消息传递进行协作。这种架构与分布式系统中的微服务理念类似：关注点分离、故障隔离、并行扩展。

本项目基于 Claude Code 的实验性 Agent Team 功能实现。但我没有选择自建一套 Agent 系统，而是将全部领域逻辑（教学网登录 via pku3b、PDF 解析策略、LaTeX 渲染管线、教学网提交协议等）内嵌在一个 skill.md 文件中。原因是：
- **由一体化 Agentic RL 训练出的 Claude Code，工具调用被蒸馏回了模型参数，且涌现出了人类炼丹师想不到的pattern, 其内置的能力远优于手工搭建的 Agent 框架**。
- 与其维护一套需要单独配置 API key 的外部系统，不如直接利用每个人本地已配置好的 Claude Code 实例——它已经有了用户的 context，已经通过了身份验证，已经具备了代码执行环境。

目前业界对多 Agent 协作的探索仍处于早期。仅有 Kimi 的 Agent Swarm 和 Claude Code 的实验性功能提供了基础能力。

本项目的价值在于验证一个端到端的完整流程：从教学网认证到作业提交，全链路由 Agent Team 自主完成。

---

## 一个想法：从《2010》到摘星揽月

> *"它们像细菌一样繁殖，以指数速度扩张，直到覆盖整个木星表面。"*
> —— Arthur C. Clarke《2010: Odyssey Two》

我理想中的 Agent 终极形态是这样的：给主 Agent 下达一个摘星揽月的宏观命令——"点燃木星"——它能够像《2010》中的黑色巨石那样，自我复制、有序分裂、集体协作、最终汇报。巨石们无需通过中央指挥来协调，而是通过极简的本地通信和相变识别（当密度达到临界点，木星变成恒星），完成了一个行星尺度的工程。

**现状是骨感的。** Agent Team 目前甚至连明确定义都没有——业界常说的 Multi-Agent 到底是指多个独立实例的协作，还是一个主 Agent 的函数调用？实现层面也极其初级，Kimi 的 Agent Swarm 和 Claude 的实验性功能提供了基础能力，但参数层面根本没有针对"自我复制与协作"进行训练。我们甚至没有普适的评估基准（benchmark）来衡量一个 Agent 系统的"增殖效率"或"解决问题的能力"。

但这正是探索的意义所在。如果一切都已成熟，那留给我们的就只有调参的工作了。

就从让 agent 完成北大学业开始吧。

---

## 功能特性

| 功能 | 说明 |
|------|------|
| 📥 **通知自动获取** | 连接教学网，下载所有课程通知和作业附件 |
| 🤖 **Agent Team 处理** | 每门课程独立 Agent 并行处理 |
| 📝 **作业自动完成** | PDF 解析 → AI 解答 → 生成答案 → 提交 |
| 📄 **智能渲染** | Markdown 自动转换为 PDF（支持 LaTeX 公式） |
| 🚀 **一键提交** | 自动提交到教学网，无需手动操作 |

---



## Claude Code 会做什么

### 模式一：获取所有通知（默认）

当你只提供学号密码时，Claude 会：

```
1. 安装并登录 pku3b
   └─ 自动下载正确版本，使用 expect 脚本登录

2. 获取课程数据
   └─ 从教学网拉取所有课程、作业、通知

3. 召唤 Agent Team
   └─ 为每门课程创建独立 Agent（并行执行）
   └─ 每个 Agent 下载附件、生成摘要

4. 生成汇总报告
   └─ 通知摘要汇总.md（紧急任务高亮）
```

**输出结果**：
```
test/
├── 通知摘要汇总.md          # 总览：所有课程的紧急任务
├── 简明量子力学/
│   ├── 作业/                # 下载的作业 PDF
│   ├── 通知/                # 课程通知
│   ├── 资料/                # 课件资料
│   └── 通知摘要.md          # 该课程的作业列表
└── ...
```

### 模式二：完成特定作业

当你指定课程和作业时：

```
skill: autopku 简明量子力学 hw5
```

Claude 会：

```
1. 列出待交作业
   └─ 让你选择具体完成哪一次作业

2. 创建 Agent Team 执行 5 Phase 流程

   Phase 1: PDF 解析
      └─ parser agent: 代码间接提取 PDF 题目
      └─ 输出: homework_parsed.json

   Phase 2: 题目解答
      └─ solver agent: 逐题解答，参考资料
      └─ 输出: answers.json

   Phase 3: 文档生成
      └─ writer agent: Markdown 格式化
      └─ 输出: HomeworkXXXX_answer.md

   Phase 4: PDF 渲染
      └─ renderer agent: Markdown → PDF
      └─ 输出: HomeworkXXXX_answer.pdf

   Phase 5: 教学网提交
      └─ portal_submitter: 自动提交
      └─ 状态: 已完成
```

**输出结果**：
```
test/简明量子力学/
├── 作业/
│   ├── Homework202605.pdf           # 原始作业
│   └── Homework202605_answer.md     # 答案 Markdown
├── 提交/
│   └── Homework202605_answer.pdf    # 最终 PDF（已提交）
├── homework_parsed.json             # PDF 解析结果
└── answers.json                     # 答案数据
```

---

## Agent Team 结构

```
Team: autopku-team
│
├── coordinator (协调者)
│   └── 分配任务，汇总结果
│
├── pdf_parser (PDF解析器)
│   └── 代码间接解析 PDF（禁止直接读取）
│
├── solver (解题者)
│   └── 逐题解答，参考资料
│
├── writer (撰写者)
│   └── 格式化 Markdown 答案
│
├── renderer (渲染器)
│   └── Markdown → PDF（Chrome Headless）
│
└── portal_submitter (提交者)
    └── 提交到教学网
```

![Agent Team 结构](./images/agent-team.png)

---

## 用户确认机制

**作业完成前必须确认**：

1. Claude 列出该课程所有待交作业
2. 你选择具体完成哪一次
3. 二次确认后开始执行
4. 可随时取消

> 禁止自动选择最新作业，必须用户明确确认。


---

## 许可证

MIT License - 自由使用和修改

---

*Happy Coding at PKU! 🎓*
