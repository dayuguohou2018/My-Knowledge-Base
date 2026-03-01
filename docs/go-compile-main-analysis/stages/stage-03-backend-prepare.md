# 阶段 03：后端准备（Runtime 符号装载 + SSA 配置 + ITab 预编译）

## 1）核心业务流程
### 1.1 阶段目标
把前端阶段产出的语义信息转换为“后端可执行配置”：
1. runtime helper 可直接引用。
2. SSA 后端配置和缓存就绪。
3. 接口分派表（itab entries）预先计算，支撑编译期去虚。

### 1.2 详细调用链
1. `typecheck.InitRuntime()`
   1. `runtimeTypes()` 构造 runtime 声明所需类型模板。
   2. 遍历 `runtimeDecls`：
      1. `funcTag -> importfunc(ir.Pkgs.Runtime, ...)`
      2. `varTag -> importvar(ir.Pkgs.Runtime, ...)`
   3. 结果：runtime 符号可被编译器内部引用，但不暴露给用户代码。
2. `ssagen.InitConfig()`
   1. `ssa.NewTypes()` + `ssa.NewConfig(...)` 构建后端配置对象。
   2. 预生成常用指针类型并关闭 `types.NewPtrCacheEnabled`（后端禁缓存，避免额外分配）。
   3. 初始化 `ssaCaches = make([]ssa.Cache, base.Flag.LowerC)`（按 worker 数配置）。
   4. 注册 runtime helper 到 `ir.Syms.*`：
      1. panic/defer/newobject/growslice。
      2. race/msan/write barrier。
      3. 各架构特有 helper（如 amd64 `gcWriteBarrierXX`）。
   5. 按架构填充边界检查函数映射：
      1. `BoundsCheckFunc[...] = panicXXX/goPanicXXX`
      2. 32 位架构再填 `ExtendCheckFunc[...]`。
3. `reflectdata.CompileITabs()`
   1. 遍历 `itabs` 待处理项。
   2. `genfun(concrete, iface)`：
      1. `imethods(iface)` 与 `methods(concrete)` 分别取已排序方法序列。
      2. 单次线性相交匹配，得到接口每个方法对应实现符号。
   3. 写回 `itab.entries`，供 SSA 阶段 `ITabSym` 查询。

### 1.3 对源码文件做了哪些处理
1. 本阶段不处理源码文本。
2. 核心动作是“符号注册 + 配置装配 + 预计算映射”：
   1. 运行时符号注入到编译器内部命名空间。
   2. 架构策略（软浮点、ABI、边界检查）固化到配置对象。
   3. 接口方法分发表预构建。

### 1.4 阶段最终输出
1. `ssaConfig`：后端行为总配置（优化开关、架构、ABI）。
2. `ssaCaches`：并行编译的 worker 本地缓存。
3. `ir.Syms.*`：runtime helper 符号句柄池。
4. `itabs[i].entries`：接口方法偏移到实现符号的映射。

## 2）产出物分析
### 2.1 Data Pipeline
| 输入 | 核心处理 | 中间结构 | 输出 |
|---|---|---|---|
| runtime 声明表 | `InitRuntime` 导入 | `types.Type` / `ir.Name` | runtime 符号可查询 |
| 架构信息 + 编译参数 | `InitConfig` 装配 | `ssa.Config` / `ssa.Cache` | 后端配置就绪 |
| itab 请求集合 | `CompileITabs/genfun` | 方法交集序列 | `entries []*obj.LSym` |

### 2.2 关键数据结构与字段
1. `ssa.Config`：
   1. `SoftFloat/Race` 等全局后端行为开关。
   2. `ABI0/ABI1` 后续用于函数参数布局与寄存器策略。
2. `ir.Syms`：
   1. `WriteBarrier/Typedmemmove/Newobject/...` 等 helper。
   2. 后续 `walk/ssa/genssa` 直接消费。
3. `itabEntry.entries`：
   1. 每个接口方法对应具体实现符号。
   2. 供 `reflectdata.ITabSym` 在编译期做偏移解析。

### 2.3 阶段边界与不变量
1. 必须先于 `reflectdata.CompileITabs()` 初始化 `InitRuntime + InitConfig`。
2. 本阶段结束后，后端不应再依赖“惰性查找 runtime 符号”。
3. 架构分支（Wasm/amd64/32bit）在此固定，后续阶段按配置执行。

## 3）核心实体
### 3.1 Interface（领域抽象）
1. `ssa.Config`：后端全局策略对象。

### 3.2 典型领域对象
1. `ir.Pkgs.Runtime`：runtime 包命名空间。
2. `ir.Syms`：编译期 runtime helper 注册表。
3. `itab` 条目：`(concrete type, interface type) -> method symbols`。

### 3.3 领域服务
1. `typecheck.InitRuntime`：runtime 声明导入服务。
2. `ssagen.InitConfig`：后端配置组装服务。
3. `reflectdata.CompileITabs`：接口分派映射预计算服务。

## 4）设计模式与思考
### 4.1 已采用模式
1. Bootstrap：统一初始化后端依赖。
2. Registry：runtime helper 集中注册到 `ir.Syms`。
3. Strategy：按目标架构切换 panic/bounds/write-barrier 策略。

### 4.2 为什么要这样设计
1. 后端热路径不能频繁做符号解析，预注册更稳定。
2. 架构差异集中在初始化点，可显著降低后续分支复杂度。
3. itab 预计算让去虚与接口调用降级更直接。

### 4.3 如果我来设计（对比）
1. 方案：全部 helper 惰性加载。
2. 优点：初始化更轻。
3. 缺点：
   1. 热路径需要查表/锁。
   2. 调试时状态分散，时序问题更多。
4. 结论：Go 的“前置装配”更适合编译器高吞吐场景。

## 代码锚点
- `src/cmd/compile/internal/gc/main.go:266`
- `src/cmd/compile/internal/gc/main.go:267`
- `src/cmd/compile/internal/gc/main.go:273`
- `src/cmd/compile/internal/typecheck/syms.go:68`
- `src/cmd/compile/internal/ssagen/ssa.go:70`
- `src/cmd/compile/internal/ssagen/ssa.go:90`
- `src/cmd/compile/internal/ssagen/ssa.go:95`
- `src/cmd/compile/internal/reflectdata/reflect.go:1231`
- `src/cmd/compile/internal/reflectdata/reflect.go:1245`
- `src/cmd/compile/internal/reflectdata/reflect.go:1281`
