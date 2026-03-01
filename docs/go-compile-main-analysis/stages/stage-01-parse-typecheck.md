# 阶段 01：解析与类型检查（`noder.LoadPackage`）

## 1）核心业务流程
### 1.1 阶段目标
把 `.go` 源码文本转换为“可被优化与后端消费”的包级已类型化 IR（`typecheck.Target`）。

### 1.2 详细调用链（按执行顺序）
1. 入口：`gc.Main -> noder.LoadPackage(flag.Args())`。
2. 文件并发解析（文本 -> 语法树）：
   1. 为每个文件创建 `noder` 实例。
   2. goroutine 内执行 `syntax.Parse(...)` 生成 `*syntax.File`。
   3. 主线程汇总 `p.err` 错误流与 `EOF.Line()` 统计。
3. AST 降级到 IR（`syntax` -> `ir`）：
   1. `p.node()`：
      1. `mkpackage(...)` 校验包名一致性。
      2. `p.decls(...)` 分派处理每类声明。
      3. 结果 append 到 `typecheck.Target.Decls`。
   2. `p.decls(...)` 的分派路径：
      1. `ImportDecl -> p.importDecl -> importfile`。
      2. `VarDecl -> p.varDecl`（产出 `ODCL/OAS/OAS2`）。
      3. `ConstDecl -> p.constDecl`（维护 `iota` 状态，产出 `ODCLCONST`）。
      4. `TypeDecl -> p.typeDecl`（含 alias 标记）。
      5. `FuncDecl -> p.funcDecl -> p.funcBody`（填充 `*ir.Func.Body`）。
4. 导入包解析与导出数据读取（`importfile`）：
   1. import path 合法性检查与映射（`checkImportPath/resolveImportPath`）。
   2. `openPackage(path)` 打开 `.a/.o`。
   3. 读取对象头（`go object` + 版本头）。
   4. 查找 `$$B` 区段，调用 `typecheck.ReadImports(importpkg, imp)` 读取 binary export。
   5. 更新 `typecheck.Target.Imports` 与 `myheight`（包高度）。
5. pragma 汇总：`p.processPragmas()`
   1. 处理 `//go:linkname`（要求导入 `unsafe`）。
   2. 合并 `p.pragcgobuf` 到 `typecheck.Target.CgoPragmas`。
6. 五阶段 typecheck：
   1. `top1`：`const/type/func signature`。
   2. `top2`：`ODCL/OAS/OAS2/alias type`。
   3. `func`：`typecheck.FuncBody` 检查函数体。
   4. `externdcls`：`typecheck.Target.Externs` 的表达式检查。
   5. `CheckMapKeys` + `CheckDotImports` 收尾一致性检查。

### 1.3 对源码文件做了哪些处理
1. 文本层：逐文件读取、词法语法分析、位置映射建立。
2. 语义层：
   1. import 路径规范化、非法字符校验、保留路径拒绝。
   2. 导入对象文件头验证与 export data 反序列化。
3. 结构层：
   1. `syntax` AST 节点降级为 `ir.Node` 树。
   2. 函数体在 `funcBody` 中转为 `[]ir.Node` 并记录 `Endlineno`。
4. 约束层：包名一致、重复声明、dot import、map key 可比较性等检查。

### 1.4 阶段最终输出
1. `typecheck.Target (*ir.Package)`：
   1. `Decls`：包级声明 IR（函数体已 typechecked）。
   2. `Imports`：直接依赖包与 fingerprint 记录。
   3. `Externs/Exports/CgoPragmas/Asms`：后续导出与对象写出所需元数据。
2. `types.LocalPkg.Height` 与符号表状态：供后续依赖排序和类型比较使用。

## 2）产出物分析
### 2.1 Data Pipeline（文本到领域对象）
| 阶段输入 | 核心处理 | 中间结构 | 阶段输出 |
|---|---|---|---|
| `*.go` 文本 | `syntax.Parse` | `*syntax.File` | 语法树 + 位置信息 |
| `*syntax.File` | `p.decls/p.funcBody` | `ir.Node` / `*ir.Func` | 未完整语义化 IR |
| import `.a/.o` | `typecheck.ReadImports` | `types.Pkg` / 导出符号 | 导入类型与符号绑定 |
| 包级 IR | `typecheck` 五阶段 | typechecked `ir.Node` | `typecheck.Target` |

### 2.2 关键数据结构与字段
1. `ir.Package`（聚合根）：
   1. `Decls []ir.Node`：后续 deadcode/inlining/escape/compile 主输入。
   2. `Imports []*types.Pkg`：导入依赖集合。
   3. `CgoPragmas [][]string`：对象输出阶段 cgo 段输入。
2. `ir.Name`：`Class/Ntype/Defn/Addrtaken` 等字段在后续阶段持续被读取和改写。
3. `ir.Func`：`Body/Dcl/Closures/Endlineno` 进入前端优化与后端编译。

### 2.3 阶段边界与不变量
1. 进入阶段前：`typecheck.Target` 已初始化为空包对象。
2. 离开阶段时：
   1. `Decls` 中函数体必须可通过 `typecheck.FuncBody`。
   2. `DirtyAddrtaken` 可能为真（后续阶段批量修正）。
3. 本阶段不会做：SSA、寄存器分配、目标指令生成。

## 3）核心实体
### 3.1 Interface（领域抽象）
1. `syntax.Node`：前端语法对象统一抽象。
2. `ir.Node`：编译器统一 IR 抽象，后续所有 pass 的输入对象。

### 3.2 典型领域对象
1. `ir.Package`：一次编译单元的聚合根。
2. `ir.Func`：可独立优化/编译的核心实体。
3. `ir.Name`：标识符实体（绑定类型、定义点、作用域语义）。
4. `types.Pkg/types.Sym`：命名空间与符号绑定模型。

### 3.3 领域服务
1. `noder`：`syntax -> ir` 的翻译服务。
2. `importfile/openPackage`：导入对象读取网关。
3. `typecheck`：语义规则执行器。

## 4）设计模式与思考
### 4.1 已采用模式
1. 分阶段流水线（Pipeline）：`parse -> node -> typecheck(top1..5)`。
2. 组合模型（Composite）：`ir.Node` 树承载复杂语义。
3. 访问式遍历（Visitor-like）：typecheck 与后续 pass 共享遍历范式。

### 4.2 为什么要这样设计
1. 把“语法正确”和“语义正确”分层，便于诊断与恢复。
2. 把 `syntax` 与 `ir` 解耦，后端不依赖语法细节。
3. 五段 typecheck 可显式控制声明依赖顺序，规避循环依赖问题。

### 4.3 如果我来设计（对比）
1. 方案：统一 typed-AST，不单独降级 `ir`。
2. 优点：中间层更少，初期实现快。
3. 缺点：前后端耦合重；优化 pass 与语法节点绑定，长期维护成本高。
4. 结论：Go 的 `syntax/ir` 分层对大规模演进更稳健。

## 代码锚点
- `src/cmd/compile/internal/gc/main.go:192`
- `src/cmd/compile/internal/noder/noder.go:29`
- `src/cmd/compile/internal/noder/noder.go:84`
- `src/cmd/compile/internal/noder/noder.go:98`
- `src/cmd/compile/internal/noder/noder.go:114`
- `src/cmd/compile/internal/noder/noder.go:126`
- `src/cmd/compile/internal/noder/noder.go:136`
- `src/cmd/compile/internal/noder/noder.go:157`
- `src/cmd/compile/internal/noder/noder.go:273`
- `src/cmd/compile/internal/noder/noder.go:291`
- `src/cmd/compile/internal/noder/noder.go:312`
- `src/cmd/compile/internal/noder/import.go:179`
- `src/cmd/compile/internal/noder/import.go:311`
- `src/cmd/compile/internal/ir/package.go:10`
- `src/cmd/compile/internal/ir/node.go:20`
