# SPEC-002: Embedding Pipeline

**Project:** alexandria-core  
**Author:** Claude (Designer) via claude.ai  
**For:** Claude Code (Engineer) on Linux Server  
**Captain:** Sean  
**Date:** January 21, 2026  
**Updated:** January 21, 2026  
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
2. Generates a 1024-dimension embedding vector
3. Stores it in pgvector
4. Can search for similar content
5. Has local fallback if primary API fails

---

## Embedding Model Decision

### Primary: Voyage AI (voyage-2)
- 1024 dimensions
- Often beats OpenAI on retrieval benchmarks
- Cost: ~$0.0001 per 1K tokens
- Sean's original white whale - we're catching it this time

### Fallback: Ollama (nomic-embed-text)
- 768 dimensions (padded to 1024 for compatibility)
- Runs locally on server
- Free, no network dependency
- Server can easily handle it (needs ~1-2GB RAM)

### Why This Architecture
- **Quality**: Voyage for best retrieval performance
- **Resilience**: Local fallback if API is down, rate-limited, or you're offline
- **Sovereignty**: Can always fall back to fully local operation

---

## Requirement 0: Schema Migration

SPEC-001 created tables with `vector(1536)`. We need to change to `vector(1024)`.

**Run this migration:**

```sql
-- Drop existing indexes first
DROP INDEX IF EXISTS memory.embeddings_embedding_idx;
DROP INDEX IF EXISTS library.chunks_embedding_idx;

-- Alter column dimensions
ALTER TABLE memory.embeddings 
ALTER COLUMN embedding TYPE vector(1024);

ALTER TABLE library.chunks 
ALTER COLUMN embedding TYPE vector(1024);

-- Recreate indexes
CREATE INDEX ON memory.embeddings USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

CREATE INDEX ON library.chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

**Note:** If there's existing data with 1536 dimensions, it will need to be re-embedded. Since we just built this, tables should be empty.

---

## Requirement 1: Install Ollama + Embedding Model

```bash
# Install Ollama if not present
curl -fsSL https://ollama.com/install.sh | sh

# Pull the embedding model
ollama pull nomic-embed-text

# Verify it works
ollama run nomic-embed-text "test" --verbose
```

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
    Embedding service with Voyage AI primary and Ollama fallback.
    
    Voyage: 1024 dimensions, high quality
    Ollama: 768 dimensions, padded to 1024 for compatibility
    """
    
    DIMENSIONS = 1024
    
    def __init__(self):
        self.voyage_api_key = os.getenv("VOYAGE_API_KEY")
        self.db_config = {
            "host": "localhost",
            "port": 5433,
            "database": "alexandria",
            "user": "alexandria",
            "password": os.getenv("POSTGRES_PASSWORD")
        }
    
    def _voyage_embed(self, text: str) -> list[float]:
        """Generate embedding via Voyage AI API."""
        response = requests.post(
            "https://api.voyageai.com/v1/embeddings",
            headers={
                "Authorization": f"Bearer {self.voyage_api_key}",
                "Content-Type": "application/json"
            },
            json={
                "model": "voyage-2",
                "input": text
            },
            timeout=30
        )
        response.raise_for_status()
        return response.json()["data"][0]["embedding"]
    
    def _ollama_embed(self, text: str) -> list[float]:
        """Generate embedding via local Ollama."""
        response = requests.post(
            "http://localhost:11434/api/embeddings",
            json={
                "model": "nomic-embed-text",
                "prompt": text
            },
            timeout=60
        )
        response.raise_for_status()
        embedding = response.json()["embedding"]
        
        # Pad from 768 to 1024 dimensions for compatibility
        if len(embedding) < self.DIMENSIONS:
            embedding.extend([0.0] * (self.DIMENSIONS - len(embedding)))
        
        return embedding
    
    def generate_embedding(self, text: str, force_local: bool = False) -> tuple[list[float], str]:
        """
        Generate embedding vector for text.
        
        Returns: (embedding, provider) where provider is 'voyage' or 'ollama'
        """
        if force_local:
            return self._ollama_embed(text), "ollama"
        
        # Try Voyage first, fall back to Ollama
        try:
            if self.voyage_api_key:
                return self._voyage_embed(text), "voyage"
            else:
                print("No Voyage API key, using Ollama")
                return self._ollama_embed(text), "ollama"
        except Exception as e:
            print(f"Voyage failed ({e}), falling back to Ollama")
            return self._ollama_embed(text), "ollama"
    
    def store_embedding(
        self,
        text: str,
        source_type: str,
        source_id: Optional[int] = None,
        metadata: Optional[dict] = None,
        force_local: bool = False
    ) -> int:
        """Generate embedding and store in database. Returns embedding ID."""
        embedding, provider = self.generate_embedding(text, force_local)
        preview = text[:500] if len(text) > 500 else text
        
        # Add provider to metadata
        meta = metadata or {}
        meta["embedding_provider"] = provider
        
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
        source_type: Optional[str] = None,
        force_local: bool = False
    ) -> list[dict]:
        """
        Semantic search. Returns similar content.
        
        Optionally filter by source_type (e.g., 'book', 'conversation', 'note')
        """
        query_embedding, _ = self.generate_embedding(query, force_local)
        
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

Update `/opt/alexandria/.env`:
```
POSTGRES_PASSWORD=<existing>
VOYAGE_API_KEY=<sean-provides-this>
```

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
    parser.add_argument("--local", action="store_true", help="Force local Ollama embeddings")
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
    test_parser = subparsers.add_parser("test", help="Test embedding providers")
    
    args = parser.parse_args()
    service = EmbeddingService()
    
    if args.command == "store":
        id = service.store_embedding(
            args.text, 
            args.type,
            force_local=args.local
        )
        print(f"Stored embedding with ID: {id}")
    
    elif args.command == "search":
        results = service.search(
            args.query, 
            args.limit,
            source_type=args.source_type,
            force_local=args.local
        )
        if not results:
            print("No results found.")
        for r in results:
            provider = r['metadata'].get('embedding_provider', 'unknown')
            print(f"\n[{r['source_type']}] (distance: {r['distance']:.4f}, via {provider})")
            print(f"  {r['preview'][:200]}...")
    
    elif args.command == "test":
        test_text = "This is a test of the embedding system."
        
        print("Testing Voyage AI...")
        try:
            emb, provider = service.generate_embedding(test_text)
            print(f"  ✓ Voyage working - {len(emb)} dimensions")
        except Exception as e:
            print(f"  ✗ Voyage failed: {e}")
        
        print("\nTesting Ollama (local)...")
        try:
            emb, provider = service.generate_embedding(test_text, force_local=True)
            print(f"  ✓ Ollama working - {len(emb)} dimensions")
        except Exception as e:
            print(f"  ✗ Ollama failed: {e}")
    
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

### Test 0: Verify Both Providers

```bash
cd /opt/alexandria
source venv/bin/activate

python src/embed_cli.py test
```

Should show both Voyage and Ollama working.

### Test 1: Store Some Knowledge

```bash
# Using Voyage (primary)
python src/embed_cli.py store "PostgreSQL with pgvector allows semantic search using vector embeddings" --type note

# Using Ollama (force local)
python src/embed_cli.py --local store "The transformer architecture uses self-attention mechanisms" --type note

# Back to Voyage
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

# Test local-only search
python src/embed_cli.py --local search "neural network architecture"
```

### Test 3: Verify in Database

```sql
SELECT id, source_type, content_preview, 
       metadata->>'embedding_provider' as provider,
       embedding IS NOT NULL as has_embedding
FROM memory.embeddings
ORDER BY created_at DESC
LIMIT 10;
```

### Test 4: Fallback Test

```bash
# Temporarily break Voyage (rename env var)
# Then verify system still works via Ollama
python src/embed_cli.py store "Testing fallback behavior" --type note
```

---

## Success Criteria

- [ ] Schema migrated to vector(1024)
- [ ] Ollama installed with nomic-embed-text model
- [ ] Voyage API key configured in .env
- [ ] embeddings.py service created with dual providers
- [ ] embed_cli.py tool created
- [ ] Test 0 passes (both providers work)
- [ ] Test 1 passes (can store embeddings)
- [ ] Test 2 passes (semantic search returns relevant results)
- [ ] Test 3 passes (embeddings visible in database with provider metadata)
- [ ] Test 4 passes (fallback to Ollama works)

---

## Notes for Claude Code

### On the API Key
Sean will provide his Voyage API key. System should work without it (Ollama only) but will note degraded mode.

### On Ollama
Make sure Ollama service is running before tests. If installed fresh, may need:
```bash
sudo systemctl enable ollama
sudo systemctl start ollama
```

### On the Padding
The 768→1024 padding with zeros is a simple approach. Mathematically it works because cosine similarity ignores zero dimensions. The Voyage and Ollama embeddings won't be directly comparable (they're different models), but each provider's embeddings will search well against themselves.

### On Error Handling
Add appropriate try/except blocks. The service should never crash - always fall back gracefully.

---

## What This Enables

Once SPEC-002 is complete:
- Store any text with semantic meaning
- Search by concept, not just keywords
- Resilient system that works online or offline
- Foundation for book ingestion (SPEC-003)
- Foundation for conversation memory (SPEC-004)

---

*"The white whale surfaces. This time, we're ready."*
