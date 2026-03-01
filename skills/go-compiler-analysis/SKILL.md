---
name: go-compiler-analysis
description: Analyze Go compiler code or other complex code modules by extracting main execution flow, core function call chains, data pipelines, and DDD models. Use when users ask for structured source-code reverse engineering output with Mermaid flow/class diagrams, stage breakdowns, and code anchors.
---

# Go Compiler Analysis

Use this skill to perform structured codebase reverse engineering with reusable outputs.

## Workflow

1. Confirm scope.
- Identify exact file(s), function(s), and target version/branch.
- Record the analysis objective: learning flow, architecture extraction, or tool replication.

2. Extract the main path.
- Start from the entry function (for compiler tasks, usually `Main`/pipeline orchestrator).
- Keep only core phases by default.
- Ignore initialization and error handling unless explicitly requested.

3. Drill into core functions.
- Expand each phase one level down to key function calls.
- For backend pipelines, explicitly capture IR transformation boundaries.
- Stop drilling once each phase has clear input/output semantics.

4. Build the data pipeline.
- For each core function, map `Input -> Transform -> Output`.
- Distinguish source text/AST/typed IR/SSA/machine code/object artifacts when relevant.

5. Build a DDD model.
- Identify aggregate roots, entities, value objects, domain services, and infrastructure services.
- Treat symbol tables/import data readers/build context as infrastructure concerns.

6. Produce diagrams.
- Generate Mermaid flowchart for execution stages.
- Generate Mermaid class diagram for core interfaces/entities and relations.

7. Attach evidence.
- Provide precise file anchors (`path:line`) for every major claim.
- Prioritize primary definitions over call sites.

## Go Compiler Main-First Baseline

When the target is Go compiler (`src/cmd/compile/...`), default to this baseline phase map unless source proves otherwise:

1. Parse and typecheck
- Typical anchor: `noder.LoadPackage(...)`.

2. Frontend transforms
- Typical anchors: deadcode cleanup, inlining, devirtualization, escape analysis.

3. Backend preparation
- Typical anchors: runtime/backend config and precompile setup.

4. Function compilation scheduling
- Typical anchors: enqueue + compile queue orchestration.

5. SSA construction and optimization
- Typical anchors: `buildssa` then `ssa.Compile`.

6. Machine instruction emission
- Typical anchors: `genssa` and assembler flush.

7. Object artifact emission
- Typical anchors: object/data dump routines.

Use this baseline to avoid missing backbone phases when extracting flow.

## Core Function Drill-Down Rules

Use these rules for each candidate core function:

1. Keep one-level expansion by default.
- Include direct critical callees only.
- Expand deeper only when a callee hides a representation boundary.

2. Prioritize boundary functions.
- Boundary examples: parse->AST, AST->IR, IR->SSA, SSA->machine code, symbol graph->object file.

3. Ignore support noise unless requested.
- Skip logging, metrics, panic wrappers, and validation-only hooks.

4. Preserve data-shape transitions.
- Every selected function must explain what data structure it consumes and produces.

## Output Contract

When the user does not provide a custom format, use this section order:

1. Core Flow
- Short stage list in execution order.
- Mermaid flowchart.

2. Data Pipeline
- Table with columns: `Function`, `Input`, `Output`.

3. DDD Model
- Aggregate roots and entities.
- Domain services and infrastructure services.
- Boundary/context notes.

4. UML Class Diagram
- Mermaid class diagram with inheritance/implementation/composition.

5. Design Review
- Patterns used.
- Why current design works.
- Alternative design and trade-offs.

6. Code Anchors
- Flat list of key `path:line` references.

## Adaptation Rules

- For non-compiler modules, map stages to that module's real lifecycle (for example: ingest -> normalize -> transform -> persist/emit).
- If there is no explicit IR, identify equivalent intermediate representations (DTO, graph, AST-like model).
- If multiple entry points exist, choose the one serving the user's current objective and state the choice.

## Optional References

- For report scaffolding, read `references/output-template.md`.
- For quality checks before finalizing, read `references/analysis-checklist.md`.
