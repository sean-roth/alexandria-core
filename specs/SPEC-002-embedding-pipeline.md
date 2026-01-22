# SPEC-002: Embedding Pipeline

**Project:** alexandria-core
**Author:** Claude (Designer) via claude.ai
**For:** Claude Code (Engineer) on Linux Server
**Captain:** Sean
**Date:** January 21, 2026
**Updated:** January 22, 2026
**Status:** Ready for Implementation
**Depends on:** SPEC-001 (complete)

---

## Context

SPEC-001 built the database with pgvector. Now we need the ability to actually generate embeddings from text. This is the missing piece that turns stored text into searchable knowledge.

Without this: keyword search only, "what did I learn about attention" fails if notes never used that exact word.

With this: semantic search, "what did I learn about attention" finds notes about "transformer mechanisms" and "self-attention layers."

---

## This Spec's Goal

Build a Python service that:
1. Takes text input
2. Generates a 768-dimension embedding vector via local Ollama
3. Stores it in pgvector
4. Can search for similar content

---

## Embedding Model Decision

### Primary (and only): Ollama with nomic-embed-text
- **768 dimensions**
- Runs locally on server
- Free, no API dependency
- Fully sovereign - works offline, no external services
- Server can easily handle it (~1-2GB RAM)

### Why Single-Provider Architecture

Previous revision considered Voyage AI as primary with Ollama fallback. This was rejected because:

1. **Different embedding models produce incompatible vector spaces.** A query embedded with Model A cannot meaningfully search content embedded with Model B. The cosine similarities are essentially random.

2. **"Fallback" was an illusion.** If Voyage went down after storing 10,000 documents, Ollama couldn't actually search that content - it would return results, but they'd be semantically meaningless. Silent failure is worse than loud failure.

3. **Simplicity wins.** One model means consistent behavior, predictable results, and no edge cases around provider switching.

4. **Local-first aligns with project values.** Digital Alexandria is about knowledge sovereignty. Depending on external APIs for core functionality contradicts that.

If embedding quality becomes a bottleneck later, the right fix is migrating entirely to a better model and re-embedding all content - not maintaining parallel systems.

---

## Requirement 0: Schema Migration

SPEC-001 created tables with `vector(1536)`. We need to change to `vector(768)` for nomic-embed-text.

**Run this migration:**

```sql
-- Drop existing indexes first
DROP INDEX IF EXISTS memory.embeddings_embedding_idx;
DROP INDEX IF EXISTS library.chunks_embedding_idx;

-- Alter column dimensions
ALTER TABLE memory.embeddings
ALTER COLUMN embedding TYPE vector(768);

ALTER TABLE library.chunks
ALTER COLUMN embedding TYPE vector(768);

-- Recreate indexes
CREATE INDEX ON memory.embeddings USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

CREATE INDEX ON library.chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

**Note:** Tables should be empty from fresh SPEC-001 install. If there's existing data, it will need to be re-embedded after migration.

---

## Requirement 1: Install Ollama Embedding Model

Ollama is already installed (v0.13.1). Pull the embedding model:

```bash
# Pull the embedding model
ollama pull nomic-embed-text

# Verify it works
curl http://localhost:11434/api/embeddings -d '{
  "model": "nomic-embed-text",
  "prompt": "test embedding"
}'
```

Expected response includes an `embedding` array with 768 floats.

---

## Requirement 2: Embedding Service

Create `/opt/alexandria/src/embeddings.py`:

```python
import os
import psycopg2
import requests
from typing import Optional
from psycopg2.extras import Json

class EmbeddingService:
    """
    Embedding service using local Ollama with nomic-embed-text.

    768 dimensions, runs entirely local, no external dependencies.
    """

    DIMENSIONS = 768
    OLLAMA_URL = "http://localhost:11434/api/embeddings"
    MODEL = "nomic-embed-text"

    def __init__(self):
        self.db_config = {
            "host": "localhost",
            "port": 5433,
            "database": "alexandria",
            "user": "alexandria",
            "password": os.getenv("POSTGRES_PASSWORD")
        }

    def generate_embedding(self, text: str) -> list[float]:
        """Generate embedding vector for text via Ollama."""
        response = requests.post(
            self.OLLAMA_URL,
            json={
                "model": self.MODEL,
                "prompt": text
            },
            timeout=60
        )
        response.raise_for_status()
        embedding = response.json()["embedding"]

        if len(embedding) != self.DIMENSIONS:
            raise ValueError(f"Expected {self.DIMENSIONS} dimensions, got {len(embedding)}")

        return embedding

    def store_embedding(
        self,
        text: str,
        source_type: str,
        source_id: Optional[int] = None,
        metadata: Optional[dict] = None
    ) -> int:
        """Generate embedding and store in database. Returns embedding ID."""
        embedding = self.generate_embedding(text)
        preview = text[:500] if len(text) > 500 else text

        meta = metadata or {}
        meta["model"] = self.MODEL

        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO memory.embeddings
            (source_type, source_id, content_preview, embedding, metadata)
            VALUES (%s, %s, %s, %s, %s)
            RETURNING id
        """, (
            source_type,
            source_id,
            preview,
            embedding,
            Json(meta)
        ))

        embedding_id = cursor.fetchone()[0]
        conn.commit()
        conn.close()

        return embedding_id

    def search(
        self,
        query: str,
        limit: int = 5,
        source_type: Optional[str] = None
    ) -> list[dict]:
        """
        Semantic search. Returns similar content.

        Optionally filter by source_type (e.g., 'book', 'conversation', 'note')
        """
        query_embedding = self.generate_embedding(query)

        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()

        if source_type:
            cursor.execute("""
                SELECT
                    id,
                    source_type,
                    source_id,
                    content_preview,
                    embedding <=> %s::vector AS distance,
                    metadata
                FROM memory.embeddings
                WHERE source_type = %s
                ORDER BY distance
                LIMIT %s
            """, (query_embedding, source_type, limit))
        else:
            cursor.execute("""
                SELECT
                    id,
                    source_type,
                    source_id,
                    content_preview,
                    embedding <=> %s::vector AS distance,
                    metadata
                FROM memory.embeddings
                ORDER BY distance
                LIMIT %s
            """, (query_embedding, limit))

        results = []
        for row in cursor.fetchall():
            results.append({
                "id": row[0],
                "source_type": row[1],
                "source_id": row[2],
                "preview": row[3],
                "distance": row[4],
                "metadata": row[5]
            })

        conn.close()
        return results
```

---

## Requirement 3: Environment Setup

The `.env` file already has `POSTGRES_PASSWORD`. No additional API keys needed.

Install dependencies:
```bash
source /opt/alexandria/venv/bin/activate
pip install requests python-dotenv
```

---

## Requirement 4: CLI Tool

Create `/opt/alexandria/src/embed_cli.py`:

```python
#!/usr/bin/env python3
"""CLI for embedding operations."""

import argparse
import sys
import os

# Add src to path
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

from dotenv import load_dotenv
load_dotenv("/opt/alexandria/.env")

from embeddings import EmbeddingService

def main():
    parser = argparse.ArgumentParser(description="Alexandria Embedding CLI")
    subparsers = parser.add_subparsers(dest="command")

    # Store command
    store_parser = subparsers.add_parser("store", help="Store text with embedding")
    store_parser.add_argument("text", help="Text to embed")
    store_parser.add_argument("--type", default="note", help="Source type")

    # Search command
    search_parser = subparsers.add_parser("search", help="Semantic search")
    search_parser.add_argument("query", help="Search query")
    search_parser.add_argument("--limit", type=int, default=5, help="Max results")
    search_parser.add_argument("--type", dest="source_type", help="Filter by source type")

    # Test command
    test_parser = subparsers.add_parser("test", help="Test embedding generation")

    args = parser.parse_args()
    service = EmbeddingService()

    if args.command == "store":
        id = service.store_embedding(args.text, args.type)
        print(f"Stored embedding with ID: {id}")

    elif args.command == "search":
        results = service.search(
            args.query,
            args.limit,
            source_type=args.source_type
        )
        if not results:
            print("No results found.")
        for r in results:
            print(f"\n[{r['source_type']}] (distance: {r['distance']:.4f})")
            print(f"  {r['preview'][:200]}...")

    elif args.command == "test":
        test_text = "This is a test of the embedding system."
        print(f"Testing Ollama with nomic-embed-text...")
        try:
            emb = service.generate_embedding(test_text)
            print(f"  Success: generated {len(emb)} dimensions")
        except Exception as e:
            print(f"  Failed: {e}")
            sys.exit(1)

    else:
        parser.print_help()

if __name__ == "__main__":
    main()
```

Make executable:
```bash
chmod +x /opt/alexandria/src/embed_cli.py
```

---

## Requirement 5: Validation

### Test 1: Verify Ollama Works

```bash
cd /opt/alexandria
source venv/bin/activate

python src/embed_cli.py test
```

Should show: `Success: generated 768 dimensions`

### Test 2: Store Some Knowledge

```bash
python src/embed_cli.py store "PostgreSQL with pgvector allows semantic search using vector embeddings" --type note

python src/embed_cli.py store "The transformer architecture uses self-attention mechanisms" --type note

python src/embed_cli.py store "Sean is building Digital Alexandria, a knowledge management system" --type note
```

### Test 3: Semantic Search

```bash
# Should find the transformer note
python src/embed_cli.py search "attention mechanisms in neural networks"

# Should find the pgvector note
python src/embed_cli.py search "vector database for AI"

# Should find the Alexandria note
python src/embed_cli.py search "Sean's projects"
```

### Test 4: Verify in Database

```sql
SELECT id, source_type, content_preview,
       metadata->>'model' as model,
       array_length(embedding::real[], 1) as dimensions
FROM memory.embeddings
ORDER BY created_at DESC
LIMIT 10;
```

Should show 3 rows, all with model=nomic-embed-text and dimensions=768.

---

## Success Criteria

- [ ] Schema migrated to vector(768)
- [ ] nomic-embed-text model pulled in Ollama
- [ ] embeddings.py service created
- [ ] embed_cli.py tool created
- [ ] Test 1 passes (Ollama generates 768-dim embeddings)
- [ ] Test 2 passes (can store embeddings)
- [ ] Test 3 passes (semantic search returns relevant results)
- [ ] Test 4 passes (embeddings visible in database with correct dimensions)

---

## Notes for Claude Code

### On Ollama
Make sure Ollama service is running before tests:
```bash
sudo systemctl status ollama
# If not running:
sudo systemctl start ollama
```

### On Error Handling
The service should fail loudly if Ollama is unavailable. No silent degradation - if embedding fails, the operation fails. This is intentional.

### On Database Port
Alexandria uses port 5433 (not default 5432) because compel-postgres already uses 5432.

---

## What This Enables

Once SPEC-002 is complete:
- Store any text with semantic meaning
- Search by concept, not just keywords
- Fully local, fully sovereign - no external dependencies
- Foundation for book ingestion (SPEC-003)
- Foundation for conversation memory (SPEC-004)

---

## Revision History

- **2026-01-21**: Initial spec with Voyage AI primary + Ollama fallback
- **2026-01-22**: Revised to Ollama-only architecture after identifying that mixed-provider embeddings produce incompatible vector spaces, making "fallback" semantically meaningless

---

*"The library runs on its own power."*
