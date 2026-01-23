# SPEC-003: Book Ingestion Pipeline

**Project:** alexandria-core  
**Author:** Claude (Designer) via claude.ai  
**For:** Claude Code (Engineer) on Linux Server  
**Captain:** Sean  
**Date:** January 23, 2026  
**Status:** Ready for Implementation  
**Depends on:** SPEC-002 (complete)

---

## Context

SPEC-002 gave us the ability to embed and search text. Now we need to turn actual documents into searchable knowledge. Sean has 17 technical PDFs ready to process - books on AI engineering, ML architecture, LangChain, transformers, prompt engineering, and more.

This is where the library gets its first real collection.

---

## This Spec's Goal

Build a pipeline that:
1. Extracts text from PDF files
2. Chunks text intelligently (preserving context)
3. Embeds each chunk via Ollama
4. Stores in the `library` schema
5. Makes books queryable: *"What did the LangChain book say about RAG?"*

---

## Architecture Overview

```
PDF File
    │
    ▼
┌─────────────────┐
│  Text Extraction │  (PyMuPDF)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Chunking     │  (split into ~500 token pieces with overlap)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Embedding     │  (Ollama nomic-embed-text)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Store in DB    │  (library.documents + library.chunks)
└─────────────────┘
```

---

## Requirement 1: Dependencies

```bash
source /opt/alexandria/venv/bin/activate
pip install PyMuPDF tiktoken
```

- **PyMuPDF (fitz)**: Fast, reliable PDF text extraction
- **tiktoken**: Token counting for consistent chunk sizes

---

## Requirement 2: Book Ingestion Service

Create `/opt/alexandria/src/ingest.py`:

```python
import os
import fitz  # PyMuPDF
import tiktoken
import psycopg2
from psycopg2.extras import Json
from typing import Optional
from dotenv import load_dotenv

load_dotenv("/opt/alexandria/.env")

from embeddings import EmbeddingService

class BookIngestionService:
    """
    Ingests PDF books into the Alexandria library.
    
    Process: PDF → Text → Chunks → Embeddings → Database
    """
    
    CHUNK_SIZE = 500      # Target tokens per chunk
    CHUNK_OVERLAP = 50    # Overlap tokens between chunks
    
    def __init__(self):
        self.embedding_service = EmbeddingService()
        self.tokenizer = tiktoken.get_encoding("cl100k_base")
        self.db_config = {
            "host": "localhost",
            "port": 5433,
            "database": "alexandria",
            "user": "alexandria",
            "password": os.getenv("POSTGRES_PASSWORD")
        }
    
    def extract_text_from_pdf(self, pdf_path: str) -> list[dict]:
        """
        Extract text from PDF, preserving page numbers.
        
        Returns: List of {"page": int, "text": str}
        """
        doc = fitz.open(pdf_path)
        pages = []
        
        for page_num, page in enumerate(doc, start=1):
            text = page.get_text()
            if text.strip():  # Skip empty pages
                pages.append({
                    "page": page_num,
                    "text": text
                })
        
        doc.close()
        return pages
    
    def count_tokens(self, text: str) -> int:
        """Count tokens in text."""
        return len(self.tokenizer.encode(text))
    
    def chunk_text(self, pages: list[dict]) -> list[dict]:
        """
        Split pages into chunks of ~CHUNK_SIZE tokens with overlap.
        
        Returns: List of {"text": str, "start_page": int, "end_page": int, "chunk_index": int}
        """
        chunks = []
        current_chunk = ""
        current_start_page = 1
        current_end_page = 1
        chunk_index = 0
        
        for page_data in pages:
            page_num = page_data["page"]
            page_text = page_data["text"]
            
            # Split page into paragraphs
            paragraphs = [p.strip() for p in page_text.split("\n\n") if p.strip()]
            
            for para in paragraphs:
                para_tokens = self.count_tokens(para)
                current_tokens = self.count_tokens(current_chunk)
                
                # If adding this paragraph exceeds chunk size, save current chunk
                if current_tokens + para_tokens > self.CHUNK_SIZE and current_chunk:
                    chunks.append({
                        "text": current_chunk.strip(),
                        "start_page": current_start_page,
                        "end_page": current_end_page,
                        "chunk_index": chunk_index
                    })
                    chunk_index += 1
                    
                    # Start new chunk with overlap from end of previous
                    overlap_text = self._get_overlap_text(current_chunk)
                    current_chunk = overlap_text + "\n\n" + para
                    current_start_page = current_end_page
                else:
                    if not current_chunk:
                        current_start_page = page_num
                    current_chunk += "\n\n" + para
                    current_end_page = page_num
        
        # Don't forget the last chunk
        if current_chunk.strip():
            chunks.append({
                "text": current_chunk.strip(),
                "start_page": current_start_page,
                "end_page": current_end_page,
                "chunk_index": chunk_index
            })
        
        return chunks
    
    def _get_overlap_text(self, text: str) -> str:
        """Get the last CHUNK_OVERLAP tokens worth of text."""
        tokens = self.tokenizer.encode(text)
        if len(tokens) <= self.CHUNK_OVERLAP:
            return text
        overlap_tokens = tokens[-self.CHUNK_OVERLAP:]
        return self.tokenizer.decode(overlap_tokens)
    
    def ingest_book(
        self,
        pdf_path: str,
        title: Optional[str] = None,
        metadata: Optional[dict] = None
    ) -> dict:
        """
        Ingest a PDF book into the library.
        
        Returns: {"document_id": int, "chunks_created": int, "total_tokens": int}
        """
        if not os.path.exists(pdf_path):
            raise FileNotFoundError(f"PDF not found: {pdf_path}")
        
        # Use filename as title if not provided
        if not title:
            title = os.path.splitext(os.path.basename(pdf_path))[0]
        
        print(f"Ingesting: {title}")
        
        # Step 1: Extract text
        print("  Extracting text...")
        pages = self.extract_text_from_pdf(pdf_path)
        print(f"  Found {len(pages)} pages with text")
        
        # Step 2: Chunk
        print("  Chunking...")
        chunks = self.chunk_text(pages)
        print(f"  Created {len(chunks)} chunks")
        
        # Step 3: Store document record
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        doc_metadata = metadata or {}
        doc_metadata["source_path"] = pdf_path
        doc_metadata["page_count"] = len(pages)
        doc_metadata["chunk_count"] = len(chunks)
        
        cursor.execute("""
            INSERT INTO library.documents (title, source_path, doc_type, metadata)
            VALUES (%s, %s, %s, %s)
            RETURNING id
        """, (title, pdf_path, "pdf", Json(doc_metadata)))
        
        document_id = cursor.fetchone()[0]
        conn.commit()
        print(f"  Document ID: {document_id}")
        
        # Step 4: Embed and store chunks
        print("  Embedding chunks...")
        total_tokens = 0
        
        for i, chunk in enumerate(chunks):
            # Progress indicator
            if (i + 1) % 10 == 0 or i == len(chunks) - 1:
                print(f"    Chunk {i + 1}/{len(chunks)}")
            
            chunk_tokens = self.count_tokens(chunk["text"])
            total_tokens += chunk_tokens
            
            # Generate embedding
            embedding = self.embedding_service.generate_embedding(chunk["text"])
            
            # Store chunk with embedding
            chunk_metadata = {
                "start_page": chunk["start_page"],
                "end_page": chunk["end_page"],
                "tokens": chunk_tokens,
                "model": "nomic-embed-text"
            }
            
            cursor.execute("""
                INSERT INTO library.chunks 
                (document_id, chunk_index, content, embedding, metadata)
                VALUES (%s, %s, %s, %s, %s)
            """, (
                document_id,
                chunk["chunk_index"],
                chunk["text"],
                embedding,
                Json(chunk_metadata)
            ))
        
        conn.commit()
        conn.close()
        
        print(f"  Complete! {len(chunks)} chunks, {total_tokens} tokens")
        
        return {
            "document_id": document_id,
            "chunks_created": len(chunks),
            "total_tokens": total_tokens
        }
    
    def search_books(
        self,
        query: str,
        limit: int = 5,
        document_id: Optional[int] = None
    ) -> list[dict]:
        """
        Semantic search across book chunks.
        
        Optionally filter to a specific document.
        """
        query_embedding = self.embedding_service.generate_embedding(query)
        
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        if document_id:
            cursor.execute("""
                SELECT 
                    c.id,
                    c.document_id,
                    d.title,
                    c.chunk_index,
                    c.content,
                    c.embedding <=> %s::vector AS distance,
                    c.metadata
                FROM library.chunks c
                JOIN library.documents d ON c.document_id = d.id
                WHERE c.document_id = %s
                ORDER BY distance
                LIMIT %s
            """, (query_embedding, document_id, limit))
        else:
            cursor.execute("""
                SELECT 
                    c.id,
                    c.document_id,
                    d.title,
                    c.chunk_index,
                    c.content,
                    c.embedding <=> %s::vector AS distance,
                    c.metadata
                FROM library.chunks c
                JOIN library.documents d ON c.document_id = d.id
                ORDER BY distance
                LIMIT %s
            """, (query_embedding, limit))
        
        results = []
        for row in cursor.fetchall():
            results.append({
                "chunk_id": row[0],
                "document_id": row[1],
                "title": row[2],
                "chunk_index": row[3],
                "content": row[4],
                "distance": row[5],
                "metadata": row[6]
            })
        
        conn.close()
        return results
    
    def list_documents(self) -> list[dict]:
        """List all ingested documents."""
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        cursor.execute("""
            SELECT id, title, doc_type, created_at, metadata
            FROM library.documents
            ORDER BY created_at DESC
        """)
        
        docs = []
        for row in cursor.fetchall():
            docs.append({
                "id": row[0],
                "title": row[1],
                "doc_type": row[2],
                "created_at": row[3],
                "metadata": row[4]
            })
        
        conn.close()
        return docs
```

---

## Requirement 3: CLI Tool

Create `/opt/alexandria/src/ingest_cli.py`:

```python
#!/usr/bin/env python3
"""CLI for book ingestion operations."""

import argparse
import sys
import os

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

from dotenv import load_dotenv
load_dotenv("/opt/alexandria/.env")

from ingest import BookIngestionService

def main():
    parser = argparse.ArgumentParser(description="Alexandria Book Ingestion CLI")
    subparsers = parser.add_subparsers(dest="command")
    
    # Ingest command
    ingest_parser = subparsers.add_parser("ingest", help="Ingest a PDF book")
    ingest_parser.add_argument("pdf_path", help="Path to PDF file")
    ingest_parser.add_argument("--title", help="Book title (default: filename)")
    
    # Ingest directory command
    batch_parser = subparsers.add_parser("batch", help="Ingest all PDFs in a directory")
    batch_parser.add_argument("directory", help="Directory containing PDFs")
    
    # Search command
    search_parser = subparsers.add_parser("search", help="Search across books")
    search_parser.add_argument("query", help="Search query")
    search_parser.add_argument("--limit", type=int, default=5, help="Max results")
    search_parser.add_argument("--book", type=int, help="Filter to specific document ID")
    
    # List command
    list_parser = subparsers.add_parser("list", help="List ingested documents")
    
    args = parser.parse_args()
    service = BookIngestionService()
    
    if args.command == "ingest":
        result = service.ingest_book(args.pdf_path, title=args.title)
        print(f"\nIngestion complete:")
        print(f"  Document ID: {result['document_id']}")
        print(f"  Chunks: {result['chunks_created']}")
        print(f"  Tokens: {result['total_tokens']}")
    
    elif args.command == "batch":
        if not os.path.isdir(args.directory):
            print(f"Error: {args.directory} is not a directory")
            sys.exit(1)
        
        pdfs = [f for f in os.listdir(args.directory) if f.lower().endswith('.pdf')]
        print(f"Found {len(pdfs)} PDF files\n")
        
        for pdf in sorted(pdfs):
            pdf_path = os.path.join(args.directory, pdf)
            try:
                result = service.ingest_book(pdf_path)
                print(f"  ✓ {result['chunks_created']} chunks\n")
            except Exception as e:
                print(f"  ✗ Error: {e}\n")
    
    elif args.command == "search":
        results = service.search_books(
            args.query,
            limit=args.limit,
            document_id=args.book
        )
        
        if not results:
            print("No results found.")
            return
        
        for r in results:
            pages = f"p.{r['metadata'].get('start_page', '?')}"
            if r['metadata'].get('end_page') != r['metadata'].get('start_page'):
                pages += f"-{r['metadata'].get('end_page', '?')}"
            
            print(f"\n[{r['title']}] {pages} (distance: {r['distance']:.4f})")
            print(f"  {r['content'][:300]}...")
    
    elif args.command == "list":
        docs = service.list_documents()
        
        if not docs:
            print("No documents ingested yet.")
            return
        
        print(f"{'ID':<4} {'Title':<50} {'Chunks':<8} {'Date'}")
        print("-" * 80)
        for doc in docs:
            chunks = doc['metadata'].get('chunk_count', '?')
            date = doc['created_at'].strftime('%Y-%m-%d') if doc['created_at'] else '?'
            title = doc['title'][:47] + "..." if len(doc['title']) > 50 else doc['title']
            print(f"{doc['id']:<4} {title:<50} {chunks:<8} {date}")
    
    else:
        parser.print_help()

if __name__ == "__main__":
    main()
```

Make executable:
```bash
chmod +x /opt/alexandria/src/ingest_cli.py
```

---

## Requirement 4: Book Storage Location

Create a directory for books on the server:

```bash
sudo mkdir -p /opt/alexandria/books
sudo chown sean:sean /opt/alexandria/books
```

Sean will need to transfer PDFs to this location via SCP, SFTP, or Syncthing.

---

## Requirement 5: Validation

### Test 1: Single Book Ingestion

Transfer one test PDF to the server, then:

```bash
cd /opt/alexandria
source venv/bin/activate

python src/ingest_cli.py ingest /opt/alexandria/books/some-book.pdf
```

Should show progress and complete without errors.

### Test 2: Search the Book

```bash
python src/ingest_cli.py search "retrieval augmented generation"
```

Should return relevant chunks from the ingested book.

### Test 3: List Documents

```bash
python src/ingest_cli.py list
```

Should show the ingested document with chunk count.

### Test 4: Verify in Database

```sql
-- Check documents
SELECT id, title, metadata->>'chunk_count' as chunks 
FROM library.documents;

-- Check chunks
SELECT document_id, COUNT(*) as chunk_count, 
       AVG(array_length(embedding::real[], 1)) as avg_dims
FROM library.chunks
GROUP BY document_id;

-- Sample chunk
SELECT content, metadata 
FROM library.chunks 
WHERE document_id = 1 
LIMIT 1;
```

### Test 5: Batch Ingestion (when ready)

```bash
python src/ingest_cli.py batch /opt/alexandria/books/
```

Should process all PDFs in the directory.

---

## Success Criteria

- [ ] PyMuPDF and tiktoken installed
- [ ] `/opt/alexandria/src/ingest.py` created
- [ ] `/opt/alexandria/src/ingest_cli.py` created
- [ ] `/opt/alexandria/books/` directory created
- [ ] Test 1 passes (single book ingestion works)
- [ ] Test 2 passes (semantic search finds relevant content)
- [ ] Test 3 passes (list shows ingested documents)
- [ ] Test 4 passes (database has correct data)

---

## Notes for Claude Code

### On Chunking Strategy
- 500 tokens is a balance between context and granularity
- 50 token overlap prevents losing context at chunk boundaries
- Paragraph-based splitting preserves semantic units

### On Performance
- Embedding is the bottleneck (~1-2 seconds per chunk with Ollama)
- A 300-page book might have 200+ chunks = 3-5 minutes to ingest
- That's fine for batch processing; we're not optimizing for real-time

### On Error Handling
- If a PDF fails, log the error and continue with batch
- Some PDFs have weird encoding or are image-based (OCR not included in this spec)

### On the Library Schema
- `library.documents`: One row per book
- `library.chunks`: Many rows per book (one per chunk)
- Chunks have embeddings; documents don't (you search chunks, not whole books)

---

## What This Enables

Once SPEC-003 is complete:
- Query any ingested book by concept
- "What did the AI Engineering book say about evaluation?"
- "Compare how different books discuss prompt engineering"
- Cross-book semantic search
- Foundation for Clara's research capabilities

---

## Batch Ingestion Estimate

Sean's 17 books, assuming ~250 pages average:
- ~200 chunks per book × 17 books = ~3,400 chunks
- ~2 seconds per chunk = ~2 hours for full ingestion
- One-time cost, then queryable forever

---

*"The books enter the library. The library remembers."*
