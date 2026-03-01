# 阶段 02：前端优化与语义变换（Deadcode / Addrtaken / Inline / Devirtualize / Escape）

## 1）核心业务流程
### 1.1 阶段目标
在已 typecheck 的 IR 上完成“可证明正确”的前端优化与语义标注，给后端提供更稳定、更低成本的输入。

### 1.2 详细调用链（按 `gc.Main` 顺序）
1. `deadcode.Func(fn)`（逐函数）
   1. `stmts(&fn.Body)` 递归处理 `if/for/range/switch/select/block`。
   2. `expr` 在 `&&/||` 上做常量短路化简。
   3. 对 `if cond const` 删除不可能分支。
   4. 若分支末尾是 `return/tailcall/panic` 且后续无 label 依赖，裁剪语句尾部。
2. `typecheck.ComputeAddrtaken(typecheck.Target.Decls)`
   1. 遍历全包 IR，捕获 `OADDR`。
   2. 对 `ir.OuterValue(...)` 为 `ONAME` 的目标设置 `Name.Addrtaken=true`。
   3. 若是 closure var，同步回写其 `Defn` 原变量。
3. `inline.InlinePackage()`
   1. `ir.VisitFuncsBottomUp(..., func(list []*ir.Func, recursive bool){...})` 按调用图 SCC 底向上遍历。
   2. `CanInline(fn)`：
      1. 过滤 pragma 约束（`go:noinline`、`go:norace`、`go:uintptrescapes` 等）。
      2. `hairyVisitor.tooHairy` 计算内联预算（`inlineMaxBudget=80`）。
      3. 通过后将 `fn.Body`/`fn.Dcl` 克隆到 `fn.Inl`。
   3. `InlineCalls(fn)`：
      1. `inlnode` 扫描 `OCALLFUNC/OCALLMETH`。
      2. `mkinlcall` 构造 `OINLCALL`。
      3. 按上下文转回表达式/语句（`inlconv2expr/inlconv2stmt/inlconv2list`）。
4. `devirtualize.Func(fn)`
   1. `ir.VisitList(fn.Body)` 捕获 `CallExpr`。
   2. `Call(call)` 仅处理 `OCALLINTER`。
   3. 若接收者静态值是 `OCONVIFACE` 且底层非接口：
      1. 构造 `TypeAssertExpr` + `OXDOT` 计算具体方法。
      2. 改写为 `OCALLMETH`（或部分去虚仍 `OCALLINTER`）。
      3. 重新计算调用结果类型（`call.SetType(...)`）。
5. `escape.Funcs(typecheck.Target.Decls)`
   1. `VisitFuncsBottomUp(..., Batch)` 在最小递归批次分析。
   2. `Batch` 内部：
      1. `initFunc`：为 `fn.Dcl` 分配 `location`，初始化结果参数 `resultIndex`。
      2. `walkFunc`：扫描语句生成数据流边（`flow`）。
      3. `flowClosure`：决定闭包变量捕获方式（by value/by reference）。
      4. `HeapAllocReason`：对必须堆分配对象直接连到 `heapLoc`。
      5. `walkAll`：图上传播逃逸关系直到收敛。
      6. `finish`：写回 `EscHeap/EscNone`、参数泄漏 tag、transient 标记。

### 1.3 对源码文件做了哪些处理
1. 本阶段不再触碰源码文本，仅处理 `ir.Node`。
2. 处理动作分三类：
   1. 结构改写：删分支、改调用形态、插入内联体。
   2. 语义标注：`Addrtaken`、`Esc`、closure 捕获策略。
   3. 成本控制：内联预算与递归环保护。

### 1.4 阶段最终输出
1. 函数 IR 结构更“后端友好”：
   1. 常量可判定分支减少。
   2. 一部分动态调用转静态调用。
   3. 内联后调用层级变浅。
2. 存储决策稳定化：
   1. 每个对象具有逃逸结论（栈/堆）。
   2. 闭包捕获方式已定（值捕获/引用捕获）。

## 2）产出物分析
### 2.1 Data Pipeline
| 输入 | 核心算法 | 中间结构 | 输出 |
|---|---|---|---|
| typechecked `ir.Func` | `deadcode.stmts/expr` | 裁剪后的语句树 | 更短控制流 IR |
| `ir.Node` 全量 | `ComputeAddrtaken` 访问遍历 | `Addrtaken` 标记 | 地址逃逸先验信息 |
| 调用图 + 函数体 | `CanInline + InlineCalls` | `ir.Inline` / `OINLCALL` | 展开后的调用点 |
| `OCALLINTER` | `devirtualize.Call` | `TypeAssertExpr/OXDOT` | `OCALLMETH` 或部分去虚 |
| 函数组 | `escape.Batch` 图求解 | `location/hole/leaks` | `Esc*` 与参数泄漏 tag |

### 2.2 关键数据结构与字段
1. 内联：
   1. `ir.Inline{Cost,Dcl,Body}` 存在于 `fn.Inl`。
   2. `inlMap map[*ir.Func]bool` 防止内联进入递归环。
2. 去虚：
   1. `ir.CallExpr.Op` 从 `OCALLINTER` 改写。
   2. `call.Type` 重算保证后端参数/返回布局正确。
3. 逃逸：
   1. `location`：节点在逃逸图中的抽象存储位置。
   2. `hole{dst,derefs}`：表达“值流向 + 解引用层级”。
   3. `leaks`：参数到 heap/result 的最短路径编码（导出到 `Field.Note`）。

### 2.3 阶段边界与不变量
1. 输入前提：函数体已 typechecked；`ir.Node.Type` 可用。
2. 输出不变量：
   1. `Addrtaken` 可供后续增量维护。
   2. `Esc` 必须在 `walk` 前确定（影响栈帧和闭包实现）。
3. 本阶段不会做：SSA 构图、寄存器分配、指令发射。

## 3）核心实体
### 3.1 Interface（领域抽象）
1. `ir.Node`：所有 pass 的统一处理对象。

### 3.2 典型领域对象
1. `ir.Func`：pass 的函数级作用域与状态容器。
2. `ir.CallExpr`：内联/去虚改写核心节点。
3. `escape.location`：逃逸图节点。
4. `escape.hole`：流上下文（目标+解引用深度）。
5. `leaks`：跨函数 escape tag 的压缩表达。

### 3.3 领域服务
1. `deadcode`：控制流裁剪服务。
2. `inline`：调用展开服务。
3. `devirtualize`：分派去动态化服务。
4. `escape`：存储生命周期推断服务。

## 4）设计模式与思考
### 4.1 已采用模式
1. 多 Pass Pipeline：每个优化 pass 单一职责。
2. 访问者/重写器：遍历与改写分离（`Visit/EditChildren`）。
3. 图数据流分析：逃逸使用约束传播 + 固定点求解。
4. SCC 分批：递归函数组作为最小分析单元，避免无限递归和过度悲观。

### 4.2 为什么要这样设计
1. 把“结构优化”和“内存决策”分开，便于定位回归。
2. 逃逸分析以图模型表达 `&/*` 关系，比规则匹配更可扩展。
3. 内联与去虚前置，能显著降低后续 SSA 图复杂度。

### 4.3 如果我来设计（对比）
1. 方案：把 `deadcode + devirtualize + inline` 融成单一超级 pass。
2. 优点：遍历次数减少。
3. 缺点：
   1. 规则耦合严重，调试困难。
   2. 回归定位不清晰。
4. 结论：Go 当前分 pass 方案更工程化，便于长期维护和性能归因。

## 代码锚点
- `src/cmd/compile/internal/gc/main.go:203`
- `src/cmd/compile/internal/gc/main.go:215`
- `src/cmd/compile/internal/gc/main.go:229`
- `src/cmd/compile/internal/gc/main.go:235`
- `src/cmd/compile/internal/gc/main.go:253`
- `src/cmd/compile/internal/deadcode/deadcode.go:14`
- `src/cmd/compile/internal/deadcode/deadcode.go:44`
- `src/cmd/compile/internal/deadcode/deadcode.go:123`
- `src/cmd/compile/internal/typecheck/subr.go:107`
- `src/cmd/compile/internal/inline/inl.go:57`
- `src/cmd/compile/internal/inline/inl.go:79`
- `src/cmd/compile/internal/inline/inl.go:525`
- `src/cmd/compile/internal/inline/inl.go:591`
- `src/cmd/compile/internal/devirtualize/devirtualize.go:18`
- `src/cmd/compile/internal/devirtualize/devirtualize.go:28`
- `src/cmd/compile/internal/escape/escape.go:209`
- `src/cmd/compile/internal/escape/escape.go:1364`
- `src/cmd/compile/internal/escape/escape.go:1655`
- `src/cmd/compile/internal/escape/escape.go:1820`
