# 阶段 04：函数入队、IR 降级与并行编译调度

## 1）核心业务流程
### 1.1 阶段目标
把前端优化后的函数 IR 变成“可并发后端编译”的任务流，并在进入 SSA 前完成最后一轮前端降级（`walk`）。

### 1.2 详细调用链
1. `enqueueFunc(fn)`（`gc.Main` 遍历 `typecheck.Target.Decls`）
   1. 过滤不可编译目标：
      1. 空白函数名 `_`。
      2. 非平凡闭包占位（由外层函数统一处理）。
   2. `len(fn.Body)==0`（bodyless，如汇编声明）分支：
      1. `ssagen.InitLSym(fn,false)`。
      2. `types.CalcSize(fn.Type())`。
      3. `AbiForBodylessFuncStackMap -> ABIAnalyzeFuncType`。
      4. `liveness.WriteFuncMap(fn, abiInfo)`。
      5. ABI0 下 `ssagen.EmitArgInfo` + `objw.Global`。
      6. 直接返回，不入 `compilequeue`。
   3. 普通函数分支：
      1. DFS 栈 `todo := []*ir.Func{fn}`。
      2. 对 `todo` 中每个函数执行 `prepareFunc(next)`。
      3. 把 `next.Closures` 继续压栈，确保闭包也完成 `walk`。
      4. 若无新增错误，仅把顶层 `fn` 入 `compilequeue`。
2. `prepareFunc(fn)`
   1. `ssagen.InitLSym(fn,true)` 提前建符号（避免并行阶段 data race）。
   2. `types.CalcSize(fn.Type())` 固定参数偏移。
   3. 设置 `typecheck.DeclContext=PAUTO` 与 `ir.CurFunc=fn`。
   4. `walk.Walk(fn)` 执行语义降级。
   5. 恢复上下文（`CurFunc=nil`, `DeclContext=PEXTERN`）。
3. `walk.Walk(fn)`
   1. `order(fn)`：提取副作用、确定求值顺序。
   2. `walkStmtList(fn.Body)`：高层语义降到更底层节点（runtime helper 调用等）。
   3. 可选 `instrument(fn)`：`-race/-msan` 等插桩。
   4. 遍历 `fn.Dcl` 预计算局部变量大小。
4. `compileFunctions()`
   1. 调度顺序：
      1. race 模式随机化（帮助发现竞态）。
      2. 否则按 `len(fn.Body)` 降序，减少尾部慢任务。
   2. 并行策略：
      1. `-c=1`：当前 goroutine 直接执行。
      2. `-c>1`：goroutine + `workerIDs chan int` 令牌池限流。
   3. 编译执行：
      1. 对每个任务 `ssagen.Compile(fn, worker)`。
      2. 完成后递归 `compile(fn.Closures)`。
   4. 并行保护：
      1. `types.CalcSizeDisabled=true`（禁并发算宽）。
      2. `base.Ctxt.InParallel=true`。
      3. `wg.Wait()` 后恢复状态。

### 1.3 对源码文件做了哪些处理
1. 已不处理文本，仅处理函数 IR。
2. `walk` 把语义节点改写为后端可消费形式。
3. 将函数组织为任务队列，并以函数粒度并发执行后端编译。

### 1.4 阶段最终输出
1. `compilequeue` 被消费为编译任务。
2. 每个函数在进入 SSA 前完成：
   1. LSym 初始化。
   2. 参数布局固定。
   3. `walk` 降级完成。
3. 并行上下文建立完成，进入 `ssagen.Compile` 阶段。

## 2）产出物分析
### 2.1 Data Pipeline
| 输入 | 核心处理 | 中间结构 | 输出 |
|---|---|---|---|
| `*ir.Func` | `enqueueFunc` 过滤/展开闭包 | `todo/compilequeue` | 待编译函数流 |
| 函数 IR | `prepareFunc + walk.Walk` | 低层语义 IR | SSA 前置输入 |
| `compilequeue` | `compileFunctions` 并行调度 | worker + WaitGroup | `ssagen.Compile` 调用流 |

### 2.2 关键数据结构与字段
1. `compilequeue []*ir.Func`：顶层待编译队列。
2. `fn.Closures []*ir.Func`：闭包递归编译链。
3. `workerIDs chan int`：并行 worker 令牌。
4. `base.Ctxt.InParallel`：通知下游进入并行后端阶段。

### 2.3 阶段边界与不变量
1. 入阶段前：逃逸分析已完成（影响 `walk/ssa`）。
2. 出阶段后：
   1. 任何进入后端的函数都已过 `walk`。
   2. 运行中不得并发调用 `types.CalcSize`。
3. 本阶段不会做：SSA 优化细分、目标指令发射、对象落盘。

## 3）核心实体
### 3.1 Interface（领域抽象）
1. `ir.Func`：后端编译任务的基本领域实体。

### 3.2 典型领域对象
1. `compilequeue`：待编译任务集合。
2. `worker`：并行执行上下文（索引到 `ssaCaches`）。
3. `walk` 后 IR：SSA 构建输入。

### 3.3 领域服务
1. `enqueueFunc`：任务构建与前置处理入口。
2. `prepareFunc`：并发前安全处理服务。
3. `walk.Walk`：语义降级服务。
4. `compileFunctions`：任务调度与并发执行服务。

## 4）设计模式与思考
### 4.1 已采用模式
1. Work Queue：`compilequeue` 聚合任务。
2. Worker Pool：`workerIDs` 令牌池限并发。
3. Producer-Consumer：入队与执行解耦。

### 4.2 为什么要这样设计
1. 函数级编译天然并行，吞吐收益高。
2. 把 `walk` 放在并行前执行，避免共享全局状态改写冲突。
3. 按函数体大小排序可减少 straggler，缩短总耗时。

### 4.3 如果我来设计（对比）
1. 方案：DAG 调度器（显式依赖边 + 优先级）。
2. 优点：可观测性与调度策略更强。
3. 缺点：实现复杂度高，当前函数级模型收益已经足够。

## 代码锚点
- `src/cmd/compile/internal/gc/main.go:279`
- `src/cmd/compile/internal/gc/main.go:281`
- `src/cmd/compile/internal/gc/main.go:287`
- `src/cmd/compile/internal/gc/compile.go:30`
- `src/cmd/compile/internal/gc/compile.go:45`
- `src/cmd/compile/internal/gc/compile.go:61`
- `src/cmd/compile/internal/gc/compile.go:81`
- `src/cmd/compile/internal/gc/compile.go:100`
- `src/cmd/compile/internal/gc/compile.go:128`
- `src/cmd/compile/internal/gc/compile.go:160`
- `src/cmd/compile/internal/walk/walk.go:24`
- `src/cmd/compile/internal/walk/walk.go:27`
- `src/cmd/compile/internal/walk/walk.go:43`
- `src/cmd/compile/internal/walk/walk.go:49`
