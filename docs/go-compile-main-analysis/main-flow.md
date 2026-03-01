# Go 编译器 `gc.Main` 核心编译流程梳理（Go 1.17）

> 范围：`src/cmd/compile/internal/gc/main.go` 中 `Main` 主流程（只保留核心编译阶段，忽略错误处理与初始化细节）。

## 按阶段拆分文档

- `docs/go-compile-main-analysis/stages/stage-01-parse-typecheck.md`
- `docs/go-compile-main-analysis/stages/stage-02-frontend-transform.md`
- `docs/go-compile-main-analysis/stages/stage-03-backend-prepare.md`
- `docs/go-compile-main-analysis/stages/stage-04-func-scheduling-lowering.md`
- `docs/go-compile-main-analysis/stages/stage-05-ssa-build-opt.md`
- `docs/go-compile-main-analysis/stages/stage-06-machine-codegen.md`
- `docs/go-compile-main-analysis/stages/stage-07-object-emission.md`

## 核心调用顺序

```mermaid
flowchart TD
    A[noder.LoadPackage(flag.Args())\n解析 + 类型检查]
    B[deadcode.Func + typecheck.ComputeAddrtaken]
    C[inline.InlinePackage]
    D[devirtualize.Func]
    E[escape.Funcs]
    F[typecheck.InitRuntime + ssagen.InitConfig]
    G[reflectdata.CompileITabs]
    H[enqueueFunc(fn)\nprepareFunc -> walk.Walk]
    I[compileFunctions()]
    J[ssagen.Compile(fn)]
    K[buildssa(fn)]
    L[ssa.Compile(s.f)\nSSA优化/调度/寄存器分配]
    M[genssa(f, pp)]
    N[pp.Flush()\n汇编为机器码]
    O[dumpdata() + dumpobj()\n写出 .o]

    A --> B --> C --> D --> E --> F --> G --> H --> I --> J --> K --> L --> M --> N --> O
```

## Data Pipeline

| 核心函数 | 数据来源（输入） | 处理产物（输出） |
|---|---|---|
| `noder.LoadPackage` | 命令行 `.go` 文件、import 的导出信息 | `typecheck.Target` 中的已类型化 IR 声明与函数体 |
| `deadcode.Func` / `typecheck.ComputeAddrtaken` | 已类型检查的函数 IR | 清理明显死代码；补齐变量 `Addrtaken` 属性 |
| `inline.InlinePackage` | 类型化函数体 IR | 调用点被内联后的函数 IR |
| `devirtualize.Func` | 内联后的调用图/方法调用点 | 可静态确定的接口/方法调用转为直接调用 |
| `escape.Funcs` | 全包函数 IR 与调用关系 | 逃逸结论（栈/堆放置、闭包捕获/大对象搬移决策） |
| `enqueueFunc` -> `prepareFunc` -> `walk.Walk` | 前端优化后的 `*ir.Func` | “walk 后”的低级 IR（更接近后端），并进入 `compilequeue` |
| `compileFunctions` -> `ssagen.Compile` -> `buildssa` | `compilequeue` 中每个函数 | 构建 `ssa.Func`，把 IR 转为 SSA 形式 |
| `ssa.Compile` | `ssa.Func` | 目标相关 SSA（优化后、块和值顺序确定、完成 regalloc） |
| `genssa` + `pp.Flush` | 已完成 regalloc 的 SSA | `obj.Prog` 指令流并落成函数机器码/符号内容 |
| `dumpdata` + `dumpobj` | 已生成的 text/data 符号与元数据 | 编译产物 `.o`（含导出数据、机器码、重定位等） |

## 关键代码锚点

- `src/cmd/compile/internal/gc/main.go:192`
- `src/cmd/compile/internal/gc/main.go:229`
- `src/cmd/compile/internal/gc/main.go:253`
- `src/cmd/compile/internal/gc/main.go:267`
- `src/cmd/compile/internal/gc/main.go:281`
- `src/cmd/compile/internal/gc/compile.go:92`
- `src/cmd/compile/internal/gc/compile.go:153`
- `src/cmd/compile/internal/ssagen/pgen.go:164`
- `src/cmd/compile/internal/ssagen/ssa.go:399`
- `src/cmd/compile/internal/ssagen/ssa.go:642`
- `src/cmd/compile/internal/ssagen/ssa.go:6732`
- `src/cmd/compile/internal/ssagen/pgen.go:190`
- `src/cmd/compile/internal/gc/main.go:305`
