# SPEC-002: Embedding Pipeline

**Project:** alexandria-core  
**Author:** Claude (Designer) via claude.ai  
**For:** Claude Code (Engineer) on Linux Server  
**Captain:** Sean  
**Date:** January 21, 2026  
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
2. Generates a 1536-dimension embedding vector
3. Stores it in pgvector
4. Can search for similar content

---

## Embedding Model Decision

### Option A: OpenAI text-embedding-ada-002
- Pros: High quality, 1536 dimensions (matches our schema), battle-tested
- Cons: Requires API key, costs money (~$0.0001 per 1K tokens), network dependency
- Cost estimate: 17 books ≈ 5M tokens ≈ $0.50 total

### Option B: Local model (e.g., sentence-transformers)
- Pros: Free, no network, privacy
- Cons: Different dimensions (may need schema change), lower quality, CPU-bound on server

### Recommendation: Start with OpenAI ada-002

Reasoning:
- We already spec'd 1536 dimensions in SPEC-001
- Cost is negligible for our scale
- Quality matters for semantic search to feel magical
- Can always add local fallback later

Sean has OpenAI API access. Store key in `/opt/alexandria/.env` alongside Postgres password.

---

## Requirement 1: Embedding Service

Create `/opt/alexandria/src/embeddings.py`:

```python
import os
import psycopg2
from openai import OpenAI
from typing import Optional

class EmbeddingService:
    def __init__(self):
        self.client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.db_config = {
            "host": "localhost",
            "port": 5433,
            "database": "alexandria",
            "user": "alexandria",
            "password": os.getenv("POSTGRES_PASSWORD")
        }
    
    def generate_embedding(self, text: str) -> list[float]:
        """Generate embedding vector for text."""
        response = self.client.embeddings.create(
            model="text-embedding-ada-002",
            input=text
        )
        return response.data[0].embedding
    
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
            metadata or {}
        ))
        
        embedding_id = cursor.fetchone()[0]
        conn.commit()
        conn.close()
        
        return embedding_id
    
    def search(self, query: str, limit: int = 5) -> list[dict]:
        """Semantic search. Returns similar content."""
        query_embedding = self.generate_embedding(query)
        
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
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

## Requirement 2: Environment Setup

Update `/opt/alexandria/.env`:
```
POSTGRES_PASSWORD=<existing>
OPENAI_API_KEY=<sean-provides-this>
```

Install dependency:
```bash
source /opt/alexandria/venv/bin/activate
pip install openai
```

---

## Requirement 3: CLI Tool

Create `/opt/alexandria/src/embed_cli.py` for testing:

```python
#!/usr/bin/env python3
"""CLI for embedding operations."""

import argparse
import os
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
    
    args = parser.parse_args()
    service = EmbeddingService()
    
    if args.command == "store":
        id = service.store_embedding(args.text, args.type)
        print(f"Stored embedding with ID: {id}")
    
    elif args.command == "search":
        results = service.search(args.query, args.limit)
        if not results:
            print("No results found.")
        for r in results:
            print(f"\n[{r['source_type']}] (distance: {r['distance']:.4f})")
            print(f"  {r['preview'][:200]}...")
    
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

## Requirement 4: Validation

### Test 1: Store Some Knowledge

```bash
cd /opt/alexandria
source venv/bin/activate

python src/embed_cli.py store "PostgreSQL with pgvector allows semantic search using vector embeddings" --type note

python src/embed_cli.py store "The transformer architecture uses self-attention mechanisms to process sequences" --type note

python src/embed_cli.py store "Sean is building Digital Alexandria, a knowledge management system" --type note
```

### Test 2: Semantic Search

```bash
# Should find the transformer note
python src/embed_cli.py search "attention mechanisms in neural networks"

# Should find the pgvector note
python src/embed_cli.py search "vector database for AI"

# Should find the Alexandria note
python src/embed_cli.py search "Sean's projects"
```

### Test 3: Verify in Database

```sql
SELECT id, source_type, content_preview, 
       embedding IS NOT NULL as has_embedding
FROM memory.embeddings
ORDER BY created_at DESC
LIMIT 10;
```

---

## Success Criteria

- [ ] OpenAI API key configured in .env
- [ ] openai package installed in venv
- [ ] embeddings.py service created
- [ ] embed_cli.py tool created
- [ ] Test 1 passes (can store embeddings)
- [ ] Test 2 passes (semantic search returns relevant results)
- [ ] Test 3 passes (embeddings visible in database)

---

## Notes for Claude Code

### On the API Key
Sean needs to provide his OpenAI API key. If not available, leave a placeholder and note in status file.

### On Error Handling
The code above is minimal for clarity. Add try/except blocks for:
- API failures
- Database connection issues
- Invalid input

### On Testing
Run the semantic search tests multiple times with different queries. The magic moment is when a search finds something that doesn't share keywords with the query.

---

## What This Enables

Once SPEC-002 is complete:
- Store any text with semantic meaning
- Search by concept, not just keywords
- Foundation for book ingestion (SPEC-003)
- Foundation for conversation memory (SPEC-004)

---

*"The embedding is the map. The vector space is the territory."*
