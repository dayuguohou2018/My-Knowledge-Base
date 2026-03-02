# `start_kernel()` 宏观管线化分析（Macro）

## 1. 核心调用序列（Call Tree / Call Order）

> 按“核心阶段”抽取，省略错误分支和细粒度初始化细节。

1. `start_kernel()` 建立早期执行契约（关中断、Boot CPU、早期安全）
2. `setup_arch()` 注入架构启动上下文并产出 `command_line`
3. `setup_boot_config()` + `setup_command_line()` 构建命令行与 init 参数输入面
4. `parse_early_param()` + `parse_args()` 完成内核参数与 init 参数分流
5. `mm_init()` + `sched_init()` + `init_IRQ()/timekeeping_init()` 完成运行时骨架
6. `local_irq_enable()` 后继续核心子系统初始化（console/security/cgroup/acpi 等）
7. `arch_call_rest_init()` -> `rest_init()` 创建 PID1 与 `kthreadd`，进入调度态
8. `kernel_init_freeable()` 执行 SMP 后续与全量 initcall
9. `kernel_init()` 回收 `__init` 资源，状态切换到 `SYSTEM_RUNNING`，执行用户态 init

代码锚点：
- `init/main.c:848`
- `init/main.c:870`
- `init/main.c:884`
- `init/main.c:904`
- `init/main.c:979`
- `init/main.c:1061`
- `init/main.c:676`
- `init/main.c:1493`
- `init/main.c:1411`

## 2. Mermaid 流程图（Flowchart）

```mermaid
flowchart TD
    A[start_kernel] --> B[setup_arch/setup_boot_config/setup_command_line]
    B --> C[parse_early_param + parse_args]
    C --> D[mm/sched/irq/time init]
    D --> E[enable IRQ + core subsystems]
    E --> F[arch_call_rest_init]
    F --> G[rest_init]
    G --> H[kernel_thread(kernel_init, pid=1)]
    G --> I[kernel_thread(kthreadd, pid=2)]
    H --> J[kernel_init_freeable]
    J --> K[do_basic_setup -> do_initcalls]
    K --> L[kernel_init finalize]
    L --> M[run /sbin/init ...]
```

## 3. Data Pipeline（Function | Input | Output）

| Function | Input | Output |
|---|---|---|
| `setup_arch` | boot params + `boot_command_line` | 架构资源与 `command_line` |
| `setup_boot_config` | initrd bootconfig + cmdline | `extra_command_line` / `extra_init_args` |
| `setup_command_line` | `boot_command_line` + `command_line` + extra args | `saved_command_line` / `static_command_line` |
| `parse_early_param` + `parse_args` | `static_command_line` + 参数区间 | 内核参数生效，`argv_init/envp_init` 更新 |
| `mm_init` | memblock/早期页表状态 | buddy/slab/vmalloc 等内存子系统可运行 |
| `sched_init` + `init_IRQ` + `timekeeping_init` | CPU/IRQ/时钟早期状态 | 可调度、可中断、可计时运行时骨架 |
| `rest_init` | `start_kernel` 后态 | PID1 + `kthreadd` + boot CPU idle |
| `kernel_init_freeable` | initcall 段边界 + 命令行快照 | 全量 initcall 执行，rootfs 准备 |
| `kernel_init` | 前序阶段产物 | `SYSTEM_RUNNING` + exec 用户态 init |

## 4. 宏观关键数据流

1. 启动参数流
- `boot_command_line` -> `setup_boot_config` -> `setup_command_line`
- `static_command_line` -> `parse_args`（内核参数）
- `--` 后参数 -> `set_init_arg` -> `argv_init`

2. 初始化任务流
- `start_kernel`（单线程） -> `rest_init`（线程化分叉）
- `kernel_init` 负责“收敛并进入用户态”
- `kthreadd` 负责内核线程执行基础设施

3. 生命周期状态流
- `SYSTEM_BOOTING` -> `SYSTEM_SCHEDULING` -> `SYSTEM_RUNNING`

代码锚点：
- `init/main.c:141`
- `init/main.c:143`
- `init/main.c:145`
- `init/main.c:506`
- `init/main.c:531`
- `init/main.c:711`
- `init/main.c:1430`
- `include/linux/kernel.h:576`
