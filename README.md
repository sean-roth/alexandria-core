# Alexandria Core

Foundation infrastructure for Digital Alexandria - a knowledge management and AI orchestration system.

## The Vision

A unified system where:
- Knowledge compounds over time through semantic search
- AI agents can read, write, and remember
- Everything syncs to human-readable Obsidian vaults
- The system documents itself as it's built

## Team Structure

- **Sean (Captain)**: Direction, decisions, feedback
- **Claude/claude.ai (Designer)**: Architecture, specs, strategy  
- **Claude Code/server (Engineer)**: Implementation, testing, maintenance

## Communication Protocol

```
/specs/     <- Designer writes specifications here
/status/    <- Engineer reports progress here
/docs/      <- Documentation that emerges from building
/src/       <- Implementation code
```

## Current Status

See `/status/` for latest progress on active specs.

## Specs

| Spec | Title | Status |
|------|-------|--------|
| [SPEC-001](specs/SPEC-001-foundation.md) | Alexandria Core Foundation | Ready for Implementation |

## Infrastructure

- **Server**: Ubuntu, i7, 32GB RAM, 17TB storage
- **Database**: PostgreSQL 16 + pgvector (Docker)
- **Sync**: Syncthing (server <-> laptop Obsidian)

---

*"The library is not just a place of books. It is a place of connection across time."*
