# 阶段 07：对象产物输出（Export + Linker Object + Archive）

## 1）核心业务流程
### 1.1 阶段目标
把编译得到的符号、类型与导出信息序列化成工具链可消费产物：
1. 编译器导入用对象（`__.PKGDEF`）。
2. 链接器用对象（`_go_.o`）。
3. 可选汇编头（`-asmhdr`）。

### 1.2 详细调用链
1. `dumpdata()`（写对象前的数据收拢）
   1. 快照计数：`numExterns/numDecls/numExports`。
   2. 写基础全局与类型相关数据：
      1. `dumpglobls(Target.Externs)`。
      2. `reflectdata.CollectPTabs()`。
      3. `addsignats(...)`。
      4. `reflectdata.WriteRuntimeTypes()`。
      5. `reflectdata.WriteTabs()/WriteImportStrings()/WriteBasicTypes()`。
      6. `dumpembeds()`。
   3. fixed-point 循环（关键）：
      1. `WriteRuntimeTypes` 可能新增 wrapper/hash/eq 函数。
      2. 扫描 `Target.Decls[numDecls:]`，对新增 `*ir.Func` 执行 `enqueueFunc`。
      3. `compileFunctions()` 编译新函数。
      4. 再次 `WriteRuntimeTypes()`，直到 `len(Target.Decls)` 稳定。
   4. 收尾：
      1. `dumpglobls(Target.Externs[numExterns:])`。
      2. 可选生成 `go.map.zero` 符号。
      3. `staticdata.WriteFuncSyms()`。
      4. `addGCLocals()` 把 `GCArgs/GCLocals/StackObjects/ArgInfo/OpenCodedDeferInfo` 加入 data list。
   5. 一致性校验：`Exports/PTabs/ITabs` 数量不得在循环后变化。
2. `dumpobj()`（按输出模式选择）
   1. 无 `-linkobj`：同一文件写 compiler+linker 两段。
   2. 有 `-linkobj`：`-o` 仅 compiler object，`-linkobj` 仅 linker object。
3. `dumpobj1(outfile, mode)`
   1. 创建 archive，写 `!<arch>\n`。
   2. `modeCompilerObj`：
      1. `startArchiveEntry`。
      2. `dumpCompilerObj`。
      3. `finishArchiveEntry(...,"__.PKGDEF")`。
   3. `modeLinkerObj`：
      1. `startArchiveEntry`。
      2. `dumpLinkerObj`。
      3. `finishArchiveEntry(...,"_go_.o")`。
4. `dumpCompilerObj(bout)`
   1. `printObjHeader` 写 go object 头/BuildID/main 标记。
   2. `dumpexport(bout)` 写 binary export：
      1. `exporter.markObject` 从 `Target.Exports` 出发标记可达类型与内联体依赖。
      2. `inline.Inline_Flood` 确保内联依赖函数可导出。
      3. 输出 `$$B` 标记 + `typecheck.WriteExports` + 结束 `$$`。
5. `dumpLinkerObj(bout)`
   1. `printObjHeader`。
   2. 若有 `Target.CgoPragmas`，写 cgo JSON 区段。
   3. 写 `!\n` 后调用 `obj.WriteObjFile(base.Ctxt, bout)`。
6. 可选 `dumpasmhdr()`
   1. 从 `Target.Asms` 导出常量宏、结构体大小与字段偏移。

### 1.3 对源码文件做了哪些处理
1. 不处理源码文本。
2. 处理对象是：
   1. `base.Ctxt` 中 text/data 符号。
   2. `typecheck.Target` 的导出/外部声明/pragma 元数据。
3. 阶段职责是“序列化与归档”，不是优化或代码生成。

### 1.4 阶段最终输出
1. archive 内条目：
   1. `__.PKGDEF`（compiler object）。
   2. `_go_.o`（linker object）。
2. 导出数据段：binary export（`$$B ... $$`）。
3. 可选 asm 头文件。

## 2）产出物分析
### 2.1 Data Pipeline
| 输入 | 核心处理 | 中间结构 | 输出 |
|---|---|---|---|
| `Target.Externs/Decls/Exports` | `dumpdata` 汇总 + fixed-point | 新增函数/类型收敛 | 完整符号与元数据集合 |
| 导出对象图 | `dumpexport` 标记并编码 | exporter 可达类型集 | binary export 流 |
| `base.Ctxt` 符号图 | `obj.WriteObjFile` | linker object sections | `_go_.o` |
| compiler/linker 段 | `dumpobj1` 归档 | archive entries | `.a` 输出 |

### 2.2 关键数据结构与字段
1. `typecheck.Target`：
   1. `Exports`：导出对象根集合。
   2. `Externs`：全局符号候选。
   3. `CgoPragmas`：cgo 区段输入。
2. `exporter.marked map[*types.Type]bool`：避免重复遍历类型图。
3. `base.Ctxt.Text/Data`：最终对象写出源数据。

### 2.3 阶段边界与不变量
1. fixed-point 循环必须收敛（有限源码不可能产生无限新类型）。
2. 收敛后 `Exports/PTabs/ITabs` 数量应稳定，否则触发 `Fatalf`。
3. 并行编译结束后才允许 `addGCLocals` 声明全局符号。

## 3）核心实体
### 3.1 Interface（领域抽象）
1. 对象写出接口：`bio.Writer` + `obj.WriteObjFile`。

### 3.2 典型领域对象
1. `exporter`：导出可达图遍历器。
2. archive entry：`__.PKGDEF` / `_go_.o`。
3. `base.Ctxt`：链接对象上下文聚合根。

### 3.3 领域服务
1. `dumpdata`：元数据汇总与补编译协调服务。
2. `dumpexport`：导出编码服务。
3. `dumpobj1`：归档封装服务。
4. `dumpLinkerObj`：链接对象序列化服务。

## 4）设计模式与思考
### 4.1 已采用模式
1. Serializer：内部图结构编码为稳定外部格式。
2. Builder：按步骤构建最终对象（header -> export -> obj sections）。
3. Fixed-point Loop：处理“写类型引入新函数”的自举依赖。

### 4.2 为什么要这样设计
1. 编译导入需求和链接需求不同，分段写出更清晰。
2. fixed-point 处理复杂运行时类型生成场景，保证完整性。
3. archive 封装保持与工具链既有格式兼容。

### 4.3 如果我来设计（对比）
1. 方案：把 fixed-point 抽出成独立阶段（先完全收敛，再统一写出）。
2. 优点：职责边界更清晰。
3. 缺点：阶段状态同步更复杂，代码路径更长。
4. 结论：当前实现把收敛逻辑内聚在 `dumpdata`，工程成本更低。

## 代码锚点
- `src/cmd/compile/internal/gc/main.go:305`
- `src/cmd/compile/internal/gc/main.go:307`
- `src/cmd/compile/internal/gc/main.go:309`
- `src/cmd/compile/internal/gc/obj.go:41`
- `src/cmd/compile/internal/gc/obj.go:50`
- `src/cmd/compile/internal/gc/obj.go:109`
- `src/cmd/compile/internal/gc/obj.go:131`
- `src/cmd/compile/internal/gc/obj.go:138`
- `src/cmd/compile/internal/gc/obj.go:154`
- `src/cmd/compile/internal/gc/obj.go:169`
- `src/cmd/compile/internal/gc/obj.go:184`
- `src/cmd/compile/internal/gc/export.go:25`
- `src/cmd/compile/internal/gc/export.go:87`
- `src/cmd/compile/internal/gc/export.go:99`
