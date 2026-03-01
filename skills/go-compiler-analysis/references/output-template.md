# Output Template

Use this template when the user does not define a custom format.

## Core Flow

```mermaid
flowchart TD
  A[Entry] --> B[Phase 1]
  B --> C[Phase 2]
  C --> D[Phase 3]
```

## Data Pipeline

| Function | Input | Output |
|---|---|---|
| `fnA` | ... | ... |
| `fnB` | ... | ... |

## DDD Model

- Aggregate root: ...
- Entities: ...
- Value objects: ...
- Domain services: ...
- Infrastructure services: ...

## UML Class Diagram

```mermaid
classDiagram
class CoreInterface {
  <<interface>>
}
class EntityA
class EntityB
CoreInterface <|.. EntityA
EntityA *-- EntityB
```

## Design Review

- Patterns: ...
- Why: ...
- Alternative: ...
- Trade-offs: ...

## Code Anchors

- `path/to/file.go:123`
- `path/to/file.go:456`
