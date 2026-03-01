---
name: code-analysis
description: "Analyze source code with a strict two-level method: macro call-tree/data-flow pipeline and micro stage deep dive. Use for structured reverse engineering, compiler main-flow understanding, stage-by-stage artifact modeling, and domain-model extraction."
---

# Code Analysis

Use this skill for structured源码分析，输出必须可复用、可落盘、可追溯到代码锚点。

## 0) 全局上下文设定（必须）

在开始正式分析前，先用以下提示词设定角色上下文：

"你现在是一个顶级的底层系统架构师和领域专家。我正在阅读相关源码。
我们的目标是：不仅要理清编译主干流程，还要提取领域模型，最终让我能自己写出类似的基础工具。
在接下来的对话中，我会按模块发给你特定的代码片段或文件路径。请你严格按照我要求的格式进行结构化输出，不要说废话。如果你明白你的角色和任务。"

## 1) 宏观管线化分析（Macro）

策略：自顶向下，抽取调用主干和数据流。

执行规则：
- 默认忽略错误处理和初始化细节。
- 只保留代表核心阶段的函数调用。
- 给出调用顺序和数据流。

必选输出：
1. 核心调用序列（Call Tree / Call Order）
2. Mermaid 流程图（Flowchart）
3. Data Pipeline 表格（`Function | Input | Output`）

## 2) 微观阶段深挖（Micro）

策略：针对宏观流程中的单一阶段做深度剖析。

每个阶段必须包含以下 4 部分：
1. 核心业务流程
- 该阶段主要工作是什么
- 对源码/IR做了哪些处理
- 详细调用链（函数级）
- 最终输出了什么

2. 产出物分析
- 输入数据形态 -> 中间结构 -> 输出结构
- 关键数据结构与核心字段

3. 核心实体
- 最重要的 Interface
- 典型领域对象（实体/值对象/服务）
- 角色分工

4. 设计模式与思考
- 采用了哪些模式（Pipeline/Visitor/Composite/Builder 等）
- 为什么这样设计
- 你的替代方案与优劣对比

## 3) 图示要求（新增）

除了宏观流程图，微观阶段还要补充阶段时序图：
- 每个阶段提供 1 个 Mermaid `sequenceDiagram`
- 时序图至少覆盖：入口函数、核心子流程、阶段输出
- 时序图可放在阶段文档内，或单独放在 `sequence-diagrams/` 目录

## 4) 文件化输出规范

当用户要求落盘文档时：
- 微观分析必须“一阶段一个 Markdown 文件”
- 推荐命名：`stage-XX-<name>.md`
- 推荐目录：`docs/<topic>/stages/`
- 若包含时序图单独文件，放在 `docs/<topic>/stages/sequence-diagrams/`
- 必须附代码锚点（`path:line`）

## 5) 质量门禁（交付前自检）

逐项确认：
- 是否覆盖宏观调用主干
- 是否每个阶段都写清“主要工作/处理过程/输出产物”
- 是否给出数据结构级别的产出物描述
- 是否提取了核心接口与典型领域对象
- 是否给出设计模式与替代方案对比
- 是否补充了阶段级 Mermaid 时序图
- 是否提供了关键代码锚点

## 6) 同步与发布（按需）

仅当用户明确要求同步到知识库/GitHub时执行：
1. 同步分析文档目录到目标仓库
2. 同步本 skill（`SKILL.md` 与相关资源）到目标仓库 `skills/`
3. 在目标仓库执行 `git add/commit/push`
4. 汇报提交哈希与变更摘要

## 参考模板

- `references/macro-template.md`
- `references/micro-template.md`
