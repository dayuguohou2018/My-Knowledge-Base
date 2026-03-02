# Linux 5.10.29 `start_kernel()` 分析文档

## 全局上下文设定

> 你现在是一个顶级的底层系统架构师和领域专家。我正在阅读相关源码。我们的目标是：不仅要理清编译主干流程，还要提取领域模型，最终让我能自己写出类似的基础工具。在接下来的对话中，我会按模块发给你特定的代码片段或文件路径。请你严格按照我要求的格式进行结构化输出，不要说废话。如果你明白你的角色和任务。

## 文档结构

- `macro-analysis.md`：宏观调用主干、流程图、数据管线
- `stages/stage-01-boot-contract-and-commandline.md`
- `stages/stage-02-parameter-parse-and-policy.md`
- `stages/stage-03-runtime-skeleton-bootstrap.md`
- `stages/stage-04-handoff-and-thread-model.md`
- `stages/stage-05-initcalls-to-userspace-init.md`

## 分析边界

- 主入口：`init/main.c:start_kernel`
- 关注范围：从 `start_kernel()` 到 `kernel_init()` 执行用户态 init
- 架构示例：x86_64 路径 `x86_64_start_kernel -> start_kernel`

关键锚点：
- `init/main.c:848`
- `init/main.c:676`
- `init/main.c:1411`
- `arch/x86/kernel/head64.c:461`
- `arch/x86/kernel/head64.c:526`
