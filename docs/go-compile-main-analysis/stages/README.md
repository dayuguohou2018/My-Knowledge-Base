# Main 函数微观阶段文档（重分析版）

以 `src/cmd/compile/internal/gc/main.go` 中 `Main` 为核心，按流程拆成 7 个阶段。
每个阶段统一包含：
1. 核心业务流程
2. 产出物分析
3. 核心实体
4. 设计模式与思考

- [stage-01-parse-typecheck.md](stage-01-parse-typecheck.md)
- [stage-02-frontend-transform.md](stage-02-frontend-transform.md)
- [stage-03-backend-prepare.md](stage-03-backend-prepare.md)
- [stage-04-func-scheduling-lowering.md](stage-04-func-scheduling-lowering.md)
- [stage-05-ssa-build-opt.md](stage-05-ssa-build-opt.md)
- [stage-06-machine-codegen.md](stage-06-machine-codegen.md)
- [stage-07-object-emission.md](stage-07-object-emission.md)

配套时序图：
- [sequence-diagrams/README.md](sequence-diagrams/README.md)

宏观管线文档：
- [../main-macro-pipeline.md](../main-macro-pipeline.md)
