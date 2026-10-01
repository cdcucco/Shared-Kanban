# Architecture decision records

`decision-register.md` is the canonical, numbered set of Phase 0 decisions (D01 to D22). Every document and every later phase follows it. Five of those decisions carry enough weight to have their own record:

| ADR | Title | Expands |
|---|---|---|
| [ADR-0001](0001-vapor-fluent-postgresql.md) | Swift, Vapor, Fluent and PostgreSQL on a Mac mini | D11 |
| [ADR-0002](0002-swiftdata-as-cache.md) | SwiftData as an offline cache only, never the shared source of truth | D12, D08 |
| [ADR-0003](0003-websockets-realtime.md) | WebSockets for realtime project events | D13, D17 |
| [ADR-0004](0004-local-attachment-storage.md) | Local attachment storage on the Mac mini | D14 |
| [ADR-0005](0005-mcp-proposal-confirmation.md) | MCP proposal and confirmation semantics | D15 |

## Changing a decision

The register is law for code and documents. To change a decision:

1. Add a new ADR (next number, same template) that states the context, the new decision, the alternatives and the consequences, and marks the superseded ADR or register entry.
2. Update the affected register entry and its Summary row in the same pull request.
3. Update every document the register entry lists as affected.

Do not edit an accepted ADR's Decision section; supersede it.

## Template

```markdown
# ADR-000N: <title>

Status: Accepted (YYYY-MM-DD)

## Context
## Decision
## Alternatives considered
## Consequences
### Positive
### Negative
### Neutral
## Verification
## Related
```
