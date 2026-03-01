# Analysis Checklist

Run this checklist before final output.

1. Scope correctness
- Entry file/function matches user request.
- Version/branch assumptions are explicit if relevant.

2. Flow completeness
- All major phases are present.
- Non-core details (init/error boilerplate) are excluded unless requested.

3. Data pipeline clarity
- Every core stage has explicit input and output.
- Representation changes are explicit (source/AST/IR/SSA/object).

4. DDD extraction quality
- Aggregate root is identified.
- Core entities and interfaces are separated from infrastructure.

5. Diagram quality
- Mermaid syntax is valid.
- Diagram nodes correspond to real functions/types.

6. Evidence quality
- Every major claim has `path:line` support.
- Anchors point to definitions when possible.

7. Communication quality
- Output is structured and concise.
- No filler text.
