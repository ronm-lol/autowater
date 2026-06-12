# Memorialization Protocol

## What gets recorded and where

| Event | Location | Format |
|-------|----------|--------|
| Architectural decision | docs/decisions/NNN-*.md | ADR template |
| Requirements change | docs/requirements.md | Timestamped changelog |
| Component selection | Relevant ADR + docs/bom.md | ADR + BOM row |
| Constraint discovered | CLAUDE.md (Key Constraints) | One-liner |
| Failed approach | Relevant ADR (status: Rejected) | ADR with failure evidence |
| Wiring/pin assignment | docs/pinout.md | Table |
| User preference | CLAUDE.md | One-liner under Conventions |

## Rules
- Never overwrite history. Append, or supersede with a new ADR.
- If a decision is reversed, the original ADR stays with status "Superseded by ADR-XXX".
- CLAUDE.md is updated only for information that changes how the agent operates. Project knowledge goes in docs/.
