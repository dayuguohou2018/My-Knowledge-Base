# 阶段 06：机器码生成（SSA -> `obj.Prog` -> 汇编文本符号）

## 1）核心业务流程
### 1.1 阶段目标
把已优化并完成寄存器分配的 `*ssa.Func` 发射为目标架构指令流，并生成 GC/调试/runtime 所需元数据。

### 1.2 详细调用链
1. 入口：`ssagen.Compile` 中的 `genssa(f, pp)`。
2. `genssa(f, pp)` 初始化
   1. 绑定 `State.ABI = f.OwnAux.Fn.ABI()`。
   2. `liveness.Compute(e.curfn, f, e.stkptrsize, pp)`：
      1. 构建变量索引与数据流框架。
      2. `prologue/solve/epilogue` 求活跃变量位图。
      3. 产出 `livenessMap`（safe point -> stack map index）。
      4. 直接向 `pp` 发 `FUNCDATA_ArgsPointerMaps/FUNCDATA_LocalsPointerMaps/(可选)StackObjects`。
   3. `emitArgInfo(e, f, pp)`：
      1. `EmitArgInfo` 编码参数布局字节流。
      2. 发 `FUNCDATA_ArgInfo`。
   4. 若函数有 open-coded defer：发 `FUNCDATA_OpenCodedDeferInfo`。
3. 基本块与值发射
   1. 记录 `bstart[blockID]`（分支回填目标）。
   2. 每块先设默认 `NextLive`（空块控制指令兜底）。
   3. `Arch.SSAMarkMoves(&s,b)` 做架构相关 move 标记。
   4. 遍历 `b.Values`：
      1. 对 `OpInitMem/OpArg/OpSP/OpSB/OpPhi/OpVarDef...` 等无代码或校验节点跳过/校验。
      2. 普通值：`s.pp.NextLive = s.livenessMap.Get(v)` 后调用 `Arch.SSAGenValue(&s,v)` 发指令。
      3. 维护 value->prog 映射（location list / HTML dump）。
   5. 块末调用 `Arch.SSAGenBlock(&s,b,next)` 发控制流指令。
4. 收尾修正
   1. `BlockExit` 末尾补 NOP，保证 panic 返回地址在函数内。
   2. open-coded defer 额外附加 `CALL deferreturn` + `RET` 段。
   3. 处理 inline mark：尽量复用已有指令位置，剩余 mark 保留为 NOP 标记。
   4. 若启用 location lists：
      1. `BuildFuncDebug/BuildFuncDebugNoOptimized`。
      2. 回填 `debugInfo.GetPC` 映射规则。
   5. 回填分支目标：`br.P.To.SetTarget(s.bstart[br.B.ID])`。
5. 返回 `Compile` 收尾
   1. 再次检查帧大小（包含 callee arg 区域）。
   2. `pp.Flush()`：汇编、模板补全、写入 text symbol。
   3. `fieldtrack` 写 `R_USEFIELD` 重定位。

### 1.3 对源码文件做了哪些处理
1. 本阶段完全不处理源码文本。
2. 处理对象是 SSA 图和符号上下文：
   1. SSA 值 -> 架构指令。
   2. safe point -> GC 位图索引。
   3. source pos -> 指令位置映射（调试信息）。

### 1.4 阶段最终输出
1. 每个函数得到 `obj.Prog` 链表并汇编为 text 符号字节。
2. 同步生成运行时元数据：
   1. `FUNCDATA`（args/locals/argInfo/stackObjects/openDefer）。
   2. `PCDATA` 与 liveness 索引。
3. 调试元数据（inline mark、location lists）准备完毕。

## 2）产出物分析
### 2.1 Data Pipeline
| 输入 | 核心处理 | 中间结构 | 输出 |
|---|---|---|---|
| `*ssa.Func` | `liveness.Compute` | `Map{ValueID->LivenessIndex}` | GC pointer map + FUNCDATA |
| ABI + 参数类型 | `EmitArgInfo` | 字节流编码 | traceback 参数描述 |
| SSA 块/值 | `Arch.SSAGenValue/Block` | `obj.Prog` 链 | 目标指令流 |
| `obj.Prog` | `pp.Flush` | 汇编器内部状态 | text symbol 机器码 |

### 2.2 关键数据结构与字段
1. `objw.Progs`：指令构建器（`Next/Prog/Flush`）。
2. `obj.Prog`：单条指令节点（含位置、目标、链接）。
3. `liveness.Map`：
   1. `Vals map[ssa.ID]LivenessIndex`。
   2. `DeferReturn`（open defer 特殊 safe point）。
4. `ssagen.State`：发射上下文（`bstart/Branches/NextLive`）。

### 2.3 阶段边界与不变量
1. 进入前：SSA pass 已完成 `regalloc` 与 `stackframe`。
2. 离开后：
   1. 所有分支目标已解析。
   2. 所有 safe point 均有可用 liveness 索引或 don't-care 语义。
3. 关键约束：`pp.Flush` 前必须完成大栈帧检查，避免汇编报错不可读。

## 3）核心实体
### 3.1 Interface（领域抽象）
1. 架构发射接口族：`Arch.SSAGenValue/SSAGenBlock/SSAMarkMoves`。

### 3.2 典型领域对象
1. `State`：代码发射过程状态机。
2. `obj.Prog`：后端统一指令 IR。
3. `liveness`：GC 安全点与变量活跃性模型。
4. `FuncDebug`：变量位置与 PC 映射。

### 3.3 领域服务
1. `liveness.Compute`：GC 位图与 safe point 计算服务。
2. `emitArgInfo`：traceback 参数布局编码服务。
3. `genssa`：SSA 到目标指令桥接服务。
4. `pp.Flush`：汇编落地服务。

## 4）设计模式与思考
### 4.1 已采用模式
1. Emitter：按 SSA op 分派到架构发射器。
2. Template Method：流程骨架固定（liveness -> emit -> branch resolve -> flush）。
3. Side-channel Metadata：指令发射同时写 FUNCDATA/调试信息。

### 4.2 为什么要这样设计
1. 编译器必须同时满足执行、GC、调试三条语义线，单一路径同步产出可防止偏差。
2. 把架构差异封装到 `Arch.*`，流程层保持稳定。
3. liveness 在发射前完成，能精确绑定 safe point 到指令。

### 4.3 如果我来设计（对比）
1. 方案：增加 Machine IR 层（SSA -> MIR -> Prog）。
2. 优点：可插入更多机器级优化。
3. 缺点：阶段变长、编译时间变高、调试链更复杂。
4. 结论：Go 当前直接发射路径在编译速度与实现复杂度上更均衡。

## 代码锚点
- `src/cmd/compile/internal/ssagen/pgen.go:164`
- `src/cmd/compile/internal/ssagen/pgen.go:175`
- `src/cmd/compile/internal/ssagen/pgen.go:190`
- `src/cmd/compile/internal/ssagen/ssa.go:6584`
- `src/cmd/compile/internal/ssagen/ssa.go:6602`
- `src/cmd/compile/internal/ssagen/ssa.go:6732`
- `src/cmd/compile/internal/ssagen/ssa.go:6738`
- `src/cmd/compile/internal/ssagen/ssa.go:6796`
- `src/cmd/compile/internal/ssagen/ssa.go:6819`
- `src/cmd/compile/internal/ssagen/ssa.go:6898`
- `src/cmd/compile/internal/ssagen/ssa.go:7014`
- `src/cmd/compile/internal/liveness/plive.go:1345`
- `src/cmd/compile/internal/liveness/plive.go:1351`
- `src/cmd/compile/internal/liveness/plive.go:1382`
