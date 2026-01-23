# SPEC-003 Progress

**Spec:** [SPEC-003-book-ingestion.md](../specs/SPEC-003-book-ingestion.md)
**Started:** 2026-01-23
**Last Updated:** 2026-01-23
**Status:** COMPLETE
**Engineer:** Claude Code

---

## Summary

Implemented the book ingestion pipeline per SPEC-003. PDFs can now be extracted, chunked, embedded via Ollama, and searched semantically. During implementation, discovered and fixed two bugs in the spec code and one issue with the Ollama embedding model.

---

## Implementation Checklist

### Dependencies
- [x] PyMuPDF 1.26.7 installed
- [x] tiktoken 0.12.0 installed

### Infrastructure
- [x] `/opt/alexandria/books/` directory created
- [x] `/opt/alexandria/src/ingest.py` created
- [x] `/opt/alexandria/src/ingest_cli.py` created (executable)

### Validation
- [x] Single book ingestion works
- [x] Semantic search returns relevant chunks
- [x] List command shows documents with metadata
- [x] Database has correct data (768-dim embeddings)

---

## Issues Encountered & Fixes

### Issue 1: Vector Formatting (Spec Bug)

**Problem:** The spec code passed raw Python lists directly to psycopg2 for pgvector columns:

```python
# Spec code (doesn't work)
embedding = self.embedding_service.generate_embedding(chunk["text"])
cursor.execute("INSERT ... VALUES (..., %s, ...)", (embedding, ...))
```

psycopg2 cannot serialize Python lists to pgvector's vector type. This causes a SQL error.

**Solution:** Format the embedding as a pgvector string literal before insertion:

```python
# Fixed code
embedding = self.embedding_service.generate_embedding(chunk["text"])
embedding_vector = "[" + ",".join(str(x) for x in embedding) + "]"
cursor.execute("INSERT ... VALUES (..., %s, ...)", (embedding_vector, ...))
```

**Rationale:** pgvector accepts vectors as string literals in the format `[0.1,0.2,...]`. This is the same fix we applied in SPEC-002's `embeddings.py`. The spec was written before we discovered this requirement during SPEC-002 implementation.

---

### Issue 2: Missing ivfflat.probes (Spec Bug)

**Problem:** The spec's `search_books()` method didn't configure the ivfflat index probes before querying. With `lists=100` in the index and few rows in the table, searches return empty or incorrect results.

**Solution:** Added probe configuration before search queries:

```python
cursor.execute("SET ivfflat.probes = 100")
```

**Rationale:** The ivfflat index partitions vectors into "lists" for approximate nearest neighbor search. By default, it only probes a small number of lists. With 100 lists configured and few documents, we need to probe all lists to find results. This is the same fix applied in SPEC-002. As the library grows with more documents, this can be tuned down for better performance.

---

### Issue 3: Ollama 500 Error with Consecutive Dots (Discovered)

**Problem:** During testing, the first PDF chunk caused Ollama to return HTTP 500:

```
requests.exceptions.HTTPError: 500 Server Error: Internal Server Error
{"error":"do embedding request: Post \"http://127.0.0.1:46385/embedding\": EOF"}
```

**Investigation:** Binary search revealed the breaking point was around 773-781 characters. The text contained table of contents formatting with many consecutive dots:

```
Contents
1. Connecting with Network Manager...................................................................1
1.1. Prevent Conflicting..................................................................................1
```

Testing confirmed: 800+ consecutive dots causes nomic-embed-text to fail; normal text of the same length works fine.

**Solution:** Added `_clean_text_for_embedding()` method that sanitizes text before embedding:

```python
def _clean_text_for_embedding(self, text: str) -> str:
    # Collapse runs of 3+ dots to ellipsis
    text = re.sub(r'\.{3,}', '...', text)
    # Collapse runs of dots with spaces
    text = re.sub(r'(\.\s*){3,}', '... ', text)
    # Normalize whitespace
    text = re.sub(r'\s+', ' ', text)
    return text.strip()
```

**Rationale:**
- Raw text is preserved in the database for retrieval display (users see original formatting)
- Only the embedding input is cleaned (semantically, `...` and `..........` are equivalent)
- This handles TOC pages, horizontal rules made of dots, and similar formatting
- The fix is defensive and won't affect normal text

---

## Usage Commands

### Transfer Books to Server

From your Windows laptop:
```bash
# Single file
scp -P 2222 "path/to/book.pdf" sean@192.168.1.205:/opt/alexandria/books/

# Multiple files
scp -P 2222 *.pdf sean@192.168.1.205:/opt/alexandria/books/

# Via Tailscale (if on VPN)
scp -P 2222 *.pdf sean@100.103.154.7:/opt/alexandria/books/
```

### Ingest Books

```bash
# Activate environment
cd /opt/alexandria
source venv/bin/activate

# Ingest single book
python src/ingest_cli.py ingest /opt/alexandria/books/some-book.pdf

# Ingest with custom title
python src/ingest_cli.py ingest /opt/alexandria/books/some-book.pdf --title "My Custom Title"

# Batch ingest all PDFs in directory
python src/ingest_cli.py batch /opt/alexandria/books/
```

### Search & List

```bash
# Search across all books
python src/ingest_cli.py search "retrieval augmented generation"

# Search with more results
python src/ingest_cli.py search "transformer architecture" --limit 10

# Search within specific book (by document ID)
python src/ingest_cli.py search "attention" --book 2

# List all ingested documents
python src/ingest_cli.py list
```

---

## Test Results

### Ingested Test Document
```
ID   Title                                              Chunks   Date
--------------------------------------------------------------------------------
2    WiFi Quick Start Guide (Test)                      4        2026-01-23
```

### Search Test
Query: `"wifi connection linux"`
```
[WiFi Quick Start Guide (Test)] p.1 (distance: 0.3669)
  system, you can use the official wpa_supplicant from Android Open
  Source Project (ASOP) to your system for Wi-Fi connection functionality...
```

Results correctly ranked by semantic relevance.

### Database Verification
```sql
SELECT document_id, COUNT(*) as chunks, AVG(array_length(embedding::real[], 1)) as dims
FROM library.chunks GROUP BY document_id;

 document_id | chunks |  dims
-------------+--------+--------
           2 |      4 |  768.0
```

---

## Performance Notes

- Embedding is the bottleneck (~1-2 seconds per chunk with Ollama on i7-4770)
- A 300-page book might have 200+ chunks = 5-10 minutes to ingest
- Batch ingestion of 17 books could take 2-4 hours
- This is a one-time cost per book; the server can run unattended

---

## Files Created

| File | Purpose |
|------|---------|
| `/opt/alexandria/src/ingest.py` | BookIngestionService class |
| `/opt/alexandria/src/ingest_cli.py` | CLI tool for ingestion operations |
| `/opt/alexandria/books/` | Storage directory for PDF files |

---

## Session Log

```
[2026-01-23 19:51] - Dependencies installed (PyMuPDF, tiktoken)
[2026-01-23 19:51] - Books directory created
[2026-01-23 19:54] - ingest.py created with spec bug fixes
[2026-01-23 19:55] - ingest_cli.py created
[2026-01-23 19:56] - First ingestion attempt failed (Ollama 500)
[2026-01-23 19:58] - Diagnosed issue: consecutive dots in TOC
[2026-01-23 19:59] - Added text cleaning for embeddings
[2026-01-23 20:00] - All validation tests passed
[2026-01-23 20:01] - SPEC-003 COMPLETE
```

---

## Recommendations for Designer

For future specs, consider:

1. **Vector formatting**: Always show explicit string formatting for pgvector inserts
2. **ivfflat.probes**: Document this requirement when using ivfflat indexes
3. **Text preprocessing**: Consider a standard text cleaning step in embedding pipelines

These are minor issues - the spec architecture and approach were sound.

---

*"The books enter the library. The library remembers."*
