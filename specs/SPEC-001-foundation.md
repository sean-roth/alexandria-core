# SPEC-001: Alexandria Core Foundation

**Project:** alexandria-core  
**Author:** Claude (Designer) via claude.ai  
**For:** Claude Code (Engineer) on Linux Server  
**Captain:** Sean  
**Date:** January 21, 2026  
**Status:** Ready for Implementation

---

## Context

Sean is building a unified knowledge management system called Digital Alexandria. This spec establishes the foundational infrastructure - the "library" before the "librarian."

### The Team Structure
- **Sean (Captain)**: Direction, decisions, feedback
- **Claude/claude.ai (Designer)**: Architecture, specs, strategy
- **Claude Code/server (Engineer)**: Implementation, testing, maintenance

### Communication Protocol
- Designer writes specs -> pushes to `/specs/` in this repo
- Engineer implements -> reports status to `/status/`
- Captain reviews, approves, redirects as needed

---

## This Spec's Goal

Set up the persistent data infrastructure that will power all future capabilities:
1. PostgreSQL with pgvector for semantic search
2. Memory schema for conversations, embeddings, decisions
3. Syncthing for Obsidian vault sync (server <-> Sean's laptop)
4. Validation that the system works from an AI's perspective

---

## Server Environment

- **OS:** Ubuntu Server
- **CPU:** Intel i7
- **RAM:** 32GB
- **Storage:** ~17TB across drives
- **Network:** Internet + ethernet hardline to Sean's Windows laptop
- **Access:** Claude Code via VS Code SSH

### First Task: Environment Audit

Before implementing, document the current state:

```bash
# Run these and save output to /status/environment-audit.md
uname -a
df -h
free -h
docker --version
docker-compose --version
python3 --version
pip3 --version
```

Note any missing dependencies.

---

## Requirement 1: PostgreSQL + pgvector

### Why PostgreSQL + pgvector
- Single database for both structured data AND vector search
- No separate vector DB to maintain
- SQL for relationships, pgvector for semantic similarity
- Simpler ops on a home server

### Docker Setup

Create `docker-compose.yml` in `/opt/alexandria/`:

```yaml
version: '3.8'

services:
  postgres:
    image: pgvector/pgvector:pg16
    container_name: alexandria-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: alexandria
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: alexandria
    volumes:
      - alexandria_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U alexandria"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  alexandria_data:
    driver: local
```

### Environment File

Create `.env` alongside docker-compose.yml:
```
POSTGRES_PASSWORD=<generate-secure-password>
```

Store the password securely. Sean will need it for future connections.

### Initialization

After container is running, connect and initialize:

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create schemas for organization
CREATE SCHEMA IF NOT EXISTS memory;    -- Conversations, context
CREATE SCHEMA IF NOT EXISTS library;   -- Books, documents, knowledge
CREATE SCHEMA IF NOT EXISTS system;    -- Metadata, configs, logs
```

---

## Requirement 2: Memory Schema

This is the "white whale" - the persistent memory that makes context compound.

### Tables in `memory` Schema

```sql
-- Conversations: Every exchange worth remembering
CREATE TABLE memory.conversations (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    participant TEXT NOT NULL,           -- 'sean', 'claude_desktop', 'claude_code', 'clara', etc.
    role TEXT NOT NULL,                  -- 'user', 'assistant', 'system'
    content TEXT NOT NULL,
    session_id UUID,                     -- Group related messages
    metadata JSONB DEFAULT '{}'          -- Flexible additional data
);

-- Embeddings: Vector representations for semantic search
CREATE TABLE memory.embeddings (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    source_type TEXT NOT NULL,           -- 'conversation', 'book', 'note', 'decision'
    source_id INTEGER,                   -- FK to source table
    content_preview TEXT,                -- First ~500 chars for display
    embedding vector(1536),              -- OpenAI ada-002 dimensions
    metadata JSONB DEFAULT '{}'
);

-- Index for fast similarity search
CREATE INDEX ON memory.embeddings USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- Decisions: Key choices with rationale (important for continuity)
CREATE TABLE memory.decisions (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    topic TEXT NOT NULL,
    decision TEXT NOT NULL,
    rationale TEXT,
    participants TEXT[],                 -- Who was involved
    supersedes INTEGER REFERENCES memory.decisions(id),
    tags TEXT[],
    metadata JSONB DEFAULT '{}'
);

-- Sessions: Group conversations into working sessions
CREATE TABLE memory.sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    started_at TIMESTAMPTZ DEFAULT NOW(),
    ended_at TIMESTAMPTZ,
    participant TEXT NOT NULL,
    summary TEXT,                        -- AI-generated session summary
    tags TEXT[],
    metadata JSONB DEFAULT '{}'
);

-- Context: Persistent context files
CREATE TABLE memory.context (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    name TEXT UNIQUE NOT NULL,           -- 'sean_profile', 'project_compel_english', etc.
    category TEXT,                       -- 'personal', 'business', 'project', 'reference'
    content TEXT NOT NULL,
    tags TEXT[],
    metadata JSONB DEFAULT '{}'
);
```

### Tables in `library` Schema (For Later)

```sql
CREATE TABLE library.documents (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    title TEXT NOT NULL,
    source_path TEXT,
    doc_type TEXT,
    metadata JSONB DEFAULT '{}'
);

CREATE TABLE library.chunks (
    id SERIAL PRIMARY KEY,
    document_id INTEGER REFERENCES library.documents(id),
    chunk_index INTEGER,
    content TEXT NOT NULL,
    embedding vector(1536),
    metadata JSONB DEFAULT '{}'
);

CREATE INDEX ON library.chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

### Tables in `system` Schema

```sql
CREATE TABLE system.config (
    key TEXT PRIMARY KEY,
    value JSONB NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE system.logs (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    level TEXT,
    component TEXT,
    message TEXT,
    metadata JSONB DEFAULT '{}'
);
```

---

## Requirement 3: Syncthing for Obsidian

### Goal
Bidirectional sync between:
- Server: `/opt/alexandria/obsidian-vault/`
- Sean's Laptop: Windows Obsidian vault

### Server Setup

```bash
sudo apt install syncthing
systemctl --user enable syncthing
systemctl --user start syncthing
```

### Vault Structure

Create on server:
```
/opt/alexandria/obsidian-vault/
├── Inbox/
├── Agents/
│   ├── Clara/
│   │   └── research-digests/
│   └── Claude/
│       └── session-summaries/
├── Library/
├── Projects/
├── Thinking/
└── System/
```

Document the Device ID in status file for Sean to connect.

---

## Requirement 4: Validation

### Test 1: Database Connection
```python
import psycopg2
conn = psycopg2.connect(
    host="localhost",
    database="alexandria",
    user="alexandria",
    password="<password>"
)
cursor = conn.cursor()
cursor.execute("SELECT version();")
print(cursor.fetchone())
```

### Test 2: Vector Operations
```python
import numpy as np
test_embedding = np.zeros(1536).tolist()
cursor.execute("""
    INSERT INTO memory.embeddings (source_type, content_preview, embedding)
    VALUES (%s, %s, %s)
    RETURNING id
""", ('test', 'This is a test entry', test_embedding))
print(f"Inserted test embedding with id: {cursor.fetchone()[0]}")
conn.commit()
```

### Test 3: Semantic Search
```python
cursor.execute("""
    SELECT id, content_preview, embedding <=> %s::vector AS distance
    FROM memory.embeddings
    ORDER BY distance
    LIMIT 5
""", (test_embedding,))
for row in cursor.fetchall():
    print(row)
```

---

## Success Criteria

- [ ] Docker running with PostgreSQL + pgvector
- [ ] All schemas created (memory, library, system)
- [ ] All tables created per schema definitions
- [ ] Vector index working
- [ ] Syncthing installed and configured
- [ ] Vault folder structure created
- [ ] Connection details documented securely
- [ ] Test scripts pass

---

## Notes for Claude Code

- If ambiguous, document your interpretation in status file
- If blocked, ask in status file
- Prefer simple, boring, reliable over clever
- Test as you go
- Update status file as you progress

---

## Next Specs (Preview)

After SPEC-001 is complete:
- **SPEC-002**: Embedding pipeline (generate embeddings for text)
- **SPEC-003**: Conversation capture (store chats automatically)
- **SPEC-004**: Book ingestion pipeline (PDF -> chunks -> embeddings)

But first: the foundation.

---

*"The library is not just a place of books. It is a place of connection across time."*
