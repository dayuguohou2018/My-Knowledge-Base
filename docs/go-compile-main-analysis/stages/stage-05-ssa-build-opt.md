# 阶段 05：SSA 构建与后端优化（`buildssa` + `ssa.Compile`）

## 1）核心业务流程
### 1.1 阶段目标
把 `walk` 后的函数 IR 转换为 SSA 图，并完成后端主优化链（含 lowering、调度、寄存器分配、栈帧布局）。

### 1.2 详细调用链
1. `ssagen.Compile(fn, worker)`
   1. `f := buildssa(fn, worker)`。
   2. 对超大栈帧做早期保护检查（>1GB 直接记录并返回）。
   3. 后续进入 `genssa`（阶段 06）。
2. `buildssa(fn, worker)`
   1. 初始化 `state` 与 `ssafn` 前端适配器。
   2. 创建 `ssa.NewFunc(&fe)` 并绑定：
      1. `Config/Cache/Name/NoSplit/ABI0/ABI1`。
      2. `ABISelf/ABIDefault`。
   3. 分配入口块与初始内存：
      1. `Entry = NewBlock(BlockPlain)`。
      2. `startmem = OpInitMem`。
      3. 创建 `OpSP/OpSB`。
   4. 处理 open-coded defer 可行性判定与相关变量。
   5. 建立局部地址映射：`decladdrs[n] = OpLocalAddr(...)`（参数/返回值）。
   6. 参数入 SSA：
      1. 可 SSA 参数：`OpArg` 放入 `s.vars`。
      2. 不可 SSA 参数：必要时从寄存器回灌到栈槽。
   7. 处理闭包变量：
      1. `OpGetClosurePtr` + 偏移寻址。
      2. 小对象按值捕获可提升为 `PAUTO` 并 SSA 化。
      3. 否则设置 heapaddr。
   8. AST-IR -> SSA-IR：
      1. `stmtList(fn.Enter)`。
      2. `zeroResults()`（返回值槽预清零，保障 defer/panic 语义）。
      3. `paramsToHeap()`（逃逸参数堆化并复制）。
      4. `stmtList(fn.Body)` + `exit()`。
   9. `insertPhis()` 插入 SSA Phi。
   10. `ssa.Compile(s.f)` 执行后端 pass 流。
   11. 记录 `RegArgs`（参数寄存器溢出信息）供后续 morestack/汇编使用。
3. `ssa.Compile(f)`
   1. 按 `passes` 顺序执行（受 `required/disabled/optimize` 控制）。
   2. 关键 pass 族：
      1. 早期清理：`phielim/copyelim/deadcode/short circuit`。
      2. 泛化优化：`opt/cse/phiopt/prove/nilcheckelim`。
      3. 语义展开：`expand calls/writebarrier/decompose`。
      4. lowering：`lower/addressing modes/checkLower`。
      5. 时序与布局：`critical/layout/schedule/tighten`。
      6. 资源分配：`flagalloc/regalloc/stackframe/trim`。
   3. pass 顺序由 `passOrder` 约束校验保证合法（例如 `schedule -> regalloc`）。

### 1.3 对源码文件做了哪些处理
1. 不处理文本；输入是 `walk` 后 `*ir.Func`。
2. 主要是 IR 形态转换和图优化：
   1. 语句树 -> SSA 图。
   2. SSA 图上多轮化简、证明、lowering、调度、分配。

### 1.4 阶段最终输出
1. `*ssa.Func` 达到“可发射”状态：
   1. 块顺序已确定（发射顺序）。
   2. 值顺序已确定（块内发射顺序）。
   3. `regalloc` 结果非空。
2. `f.RegArgs` 等 ABI 辅助信息准备完成。

## 2）产出物分析
### 2.1 Data Pipeline
| 输入 | 核心处理 | 中间结构 | 输出 |
|---|---|---|---|
| `walk` 后 `*ir.Func` | `buildssa` 图构建 | `ssa.Func/Block/Value` 初始图 | 可优化 SSA |
| 初始 SSA 图 | `ssa.Compile` pass 流 | 多次中间 SSA 形态 | 已优化 + 已分配 SSA |
| ABI 信息 | 参数/寄存器映射 | `RegArgs/OwnAux` | 汇编阶段可用元数据 |

### 2.2 关键数据结构与字段
1. `ssa.Func`：
   1. `Blocks/Entry/Config/Cache/ABIs`。
   2. `OwnAux`（调用约定信息）。
   3. `RegArgs`（参数寄存器溢出记录）。
2. `state.vars`：前端变量到 SSA 值/内存的映射。
3. `decladdrs`：源级变量到栈地址值的索引。
4. `passes`：后端优化流程声明（含 required/time/mem/debug 标志）。

### 2.3 阶段边界与不变量
1. 入阶段前：
   1. `walk` 已完成。
   2. 逃逸结论已写回 `ir.Name.Esc`。
2. 出阶段后：
   1. 每个 SSA 值映射到 0/1 条目标指令语义约束。
   2. 关键约束（`checkLower/passOrder`）已验证。
3. 本阶段不做最终指令写盘（那是 `genssa + pp.Flush` 阶段）。

## 3）核心实体
### 3.1 Interface（领域抽象）
1. `ssa.Func/ssa.Block/ssa.Value`：SSA 领域模型核心三元组。

### 3.2 典型领域对象
1. `ssafn`：前端函数到 SSA 接口的适配器。
2. `ssa.Cache`：worker 级复用缓存。
3. `pass`：后端变换单元（名称+执行函数+约束）。

### 3.3 领域服务
1. `buildssa`：图构建与前端语义映射服务。
2. `ssa.Compile`：后端 pass 编排与执行服务。
3. `regalloc/stackframe`：资源分配落地服务。

## 4）设计模式与思考
### 4.1 已采用模式
1. Builder：`buildssa` 分步搭建 `ssa.Func`。
2. Adapter：`ssafn` 屏蔽前端对象差异。
3. Multi-pass Pipeline：`passes` 顺序执行。
4. Constraint Check：`passOrder` 在 init 时强约束阶段顺序。

### 4.2 为什么要这样设计
1. SSA 图使数据依赖显式化，利于全局优化与寄存器分配。
2. pass 小步快跑，便于性能剖析和回归定位。
3. 有序约束避免“pass 调整导致隐式破坏”的工程风险。

### 4.3 如果我来设计（对比）
1. 方案：Region-based 优化框架（按区域迭代）。
2. 优点：大函数可能减少全图扫描开销。
3. 缺点：跨区域优化规则复杂，正确性成本更高。
4. 结论：Go 当前全图 + 有序 pass 方案在稳定性和可维护性上更优。

## 代码锚点
- `src/cmd/compile/internal/gc/compile.go:153`
- `src/cmd/compile/internal/ssagen/pgen.go:164`
- `src/cmd/compile/internal/ssagen/pgen.go:173`
- `src/cmd/compile/internal/ssagen/ssa.go:399`
- `src/cmd/compile/internal/ssagen/ssa.go:455`
- `src/cmd/compile/internal/ssagen/ssa.go:474`
- `src/cmd/compile/internal/ssagen/ssa.go:618`
- `src/cmd/compile/internal/ssagen/ssa.go:639`
- `src/cmd/compile/internal/ssagen/ssa.go:642`
- `src/cmd/compile/internal/ssagen/ssa.go:654`
- `src/cmd/compile/internal/ssa/compile.go:23`
- `src/cmd/compile/internal/ssa/compile.go:69`
- `src/cmd/compile/internal/ssa/compile.go:432`
- `src/cmd/compile/internal/ssa/compile.go:496`
