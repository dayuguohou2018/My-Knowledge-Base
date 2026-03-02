# Stage 04: 控制权交接与线程化启动模型

## 1. 核心业务流程

### 该阶段主要工作
- 从 `start_kernel()` 的单线程启动路径，过渡到内核线程化执行模型。
- 通过 `rest_init()` 创建 PID1（`kernel_init`）与 PID2（`kthreadd`），并让 boot CPU 进入 idle 循环。

### 对源码做了哪些处理
- `arch_call_rest_init()` 作为架构可覆写入口，默认调用 `rest_init()`。
- `rest_init()` 先创建 `kernel_init`（保证 PID1），再创建 `kthreadd`，然后 `complete(kthreadd_done)` 放行后续工作。
- 将系统状态切换到 `SYSTEM_SCHEDULING`。

### 详细调用链（函数级）
- `start_kernel()`
- `arch_call_rest_init()`
- `rest_init()`
- `kernel_thread(kernel_init, ...)`
- `kernel_thread(kthreadd, ...)`
- `schedule_preempt_disabled()`
- `cpu_startup_entry(CPUHP_ONLINE)`

### 最终输出
- 启动主线完成线程化：PID1 可推进系统收敛，`kthreadd` 可承载内核线程，boot CPU 转入 idle

## 2. 产出物分析

### 输入 -> 中间 -> 输出
- 输入：`start_kernel` 后期初始化完成态
- 中间：`kthreadd_done` completion、`kthreadd_task`、pid 绑定信息
- 输出：`SYSTEM_SCHEDULING` 状态下的多线程启动框架

### 关键数据结构与核心字段
- `DECLARE_COMPLETION(kthreadd_done)`
- `kthreadd_task`
- `system_state`

## 3. 核心实体

### 最重要的 Interface
- `void __init __weak arch_call_rest_init(void)`
- `void __ref rest_init(void)`

### 典型领域对象
- PID1 初始化线程（`kernel_init`）
- PID2 线程管理服务（`kthreadd`）
- boot idle 线程

### 角色分工
- `kernel_init`：系统最终收敛并进入用户态
- `kthreadd`：提供内核线程执行基础设施
- boot CPU：维持 idle 与调度循环

## 4. 设计模式与思考

### 采用的模式
- `Handoff + Completion Synchronization`

### 为什么这样设计
- 保证 PID1 语义稳定，并通过 completion 明确线程系统可用时点，避免启动期竞态。

### 替代方案与优劣
- 替代：先起 `kthreadd` 再起 `kernel_init`。
- 优点：线程设施更早可用。
- 缺点：破坏“init 线程必须是 PID1”的语义约束。

## 5. 阶段时序图

```mermaid
sequenceDiagram
    participant SK as start_kernel
    participant ACRI as arch_call_rest_init
    participant RI as rest_init
    participant KI as kernel_init(pid=1)
    participant KD as kthreadd(pid=2)
    participant IDLE as cpu_startup_entry
    SK->>ACRI: handoff
    ACRI->>RI: rest_init()
    RI->>KI: kernel_thread(kernel_init)
    RI->>KD: kernel_thread(kthreadd)
    RI->>RI: system_state=SYSTEM_SCHEDULING; complete(kthreadd_done)
    RI->>IDLE: enter boot cpu idle loop
```

## 6. 代码锚点

- `init/main.c:843`
- `init/main.c:845`
- `init/main.c:676`
- `init/main.c:687`
- `init/main.c:699`
- `init/main.c:711`
- `init/main.c:713`
- `init/main.c:721`
- `init/main.c:674`
- `include/linux/kernel.h:576`
- `arch/s390/kernel/setup.c:356`
