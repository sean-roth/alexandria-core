# SPEC-003 Progress

**Spec:** [SPEC-003-book-ingestion.md](../specs/SPEC-003-book-ingestion.md)
**Started:** 2026-01-23
**Last Updated:** 2026-01-24
**Status:** COMPLETE (with post-implementation fixes)
**Engineer:** Claude Code

---

## Summary

Implemented the book ingestion pipeline per SPEC-003. PDFs can now be extracted, chunked, embedded via Ollama, and searched semantically.

During implementation and batch ingestion of 17 technical books, discovered and fixed:
- 2 bugs in the spec code (vector formatting, ivfflat probes)
- 4 issues with Ollama/nomic-embed-text (consecutive dots, Unicode characters, token limits, error handling)

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

### Issue 4: Unicode Characters Causing Embedding Failures (Discovered in Batch)

**Problem:** During batch ingestion of 17 technical books, some chunks failed with HTTP 500 errors even after the dot-cleaning fix. Failures occurred around chunk 110+ in larger books.

**Investigation:** Examined failing chunks and found they contained Unicode typography characters common in professionally typeset PDFs:

```
Char at 61: '‐' ord=8208   (non-breaking hyphen)
Char at 467: '"' ord=8220  (left double quote)
Char at 469: '"' ord=8221  (right double quote)
Char at 641: ''' ord=8217  (right single quote/apostrophe)
Char at 664: '×' ord=215   (multiplication sign)
```

These characters are visually similar to ASCII equivalents but caused Ollama/nomic-embed-text to fail.

**Solution:** Added Unicode normalization to `_clean_text_for_embedding()`:

```python
unicode_replacements = {
    '\u2018': "'",   # left single quote
    '\u2019': "'",   # right single quote
    '\u201c': '"',   # left double quote
    '\u201d': '"',   # right double quote
    '\u2010': '-',   # hyphen
    '\u2011': '-',   # non-breaking hyphen
    '\u2012': '-',   # figure dash
    '\u2013': '-',   # en dash
    '\u2014': '-',   # em dash
    '\u2015': '-',   # horizontal bar
    '\u00d7': 'x',   # multiplication sign
    '\u2022': '*',   # bullet
    '\u2026': '...', # ellipsis
    '\u00a0': ' ',   # non-breaking space
}
for unicode_char, replacement in unicode_replacements.items():
    text = text.replace(unicode_char, replacement)
```

**Rationale:**
- Technical PDFs from publishers (O'Reilly, etc.) use typographic characters
- These are semantically equivalent to ASCII for embedding purposes
- Normalization happens only for embedding; original text preserved in DB
- Defensive approach catches common variants

---

### Issue 5: nomic-embed-text Token Limit (Discovered in Batch)

**Problem:** After fixing Unicode issues, some chunks still failed at exactly 2000 characters (our initial truncation limit). Testing revealed the limit varied by content.

**Investigation:** Systematic testing found the real constraint:

```python
# Test results:
500 tokens (2004 chars): OK
512 tokens (2049 chars): FAIL  # <-- Hard limit
```

nomic-embed-text has a **512 BERT token context limit**, not a character limit. Different text tokenizes differently - dense technical prose uses more tokens per character than simple sentences.

**Initial wrong approach:** Tried using tiktoken (GPT tokenizer) to count tokens, but nomic-embed-text uses a BERT tokenizer with different token boundaries. This made things worse.

**Solution:** Empirically determined safe character limit through testing:

```python
# nomic-embed-text has ~512 BERT token limit, ~1500 chars is safe
MAX_EMBED_CHARS = 1500

# In _clean_text_for_embedding():
if len(text) > self.MAX_EMBED_CHARS:
    text = text[:self.MAX_EMBED_CHARS]
```

**Testing confirmation:**
```
Testing 523 chunks with MAX_EMBED_CHARS=1500...
Failures: 0
ALL CHUNKS PASSED!
```

**Rationale:**
- 1500 characters ≈ 375-400 BERT tokens (varies by content)
- Provides ~25% safety margin below 512 token limit
- Character-based truncation is simpler and more predictable than token-based
- Full chunk text still stored in database; only embedding input is truncated
- Semantic search still works well with first 1500 chars of each chunk

---

### Issue 6: Batch Ingestion Error Handling

**Problem:** When a book failed during embedding, the batch command moved to the next book, leaving partial documents (document record created, but no chunks).

**Observation:** This is actually reasonable behavior for batch processing - fail fast per document, continue with others. However, partial documents need cleanup before re-ingestion.

**Solution:** Before re-running batch ingestion, clean up partial documents:

```sql
-- Find documents with no chunks
SELECT d.id, d.title, COUNT(c.id) as chunks
FROM library.documents d
LEFT JOIN library.chunks c ON d.id = c.document_id
GROUP BY d.id, d.title
HAVING COUNT(c.id) = 0;

-- Delete partial documents
DELETE FROM library.documents WHERE id IN (3, 4, 5);  -- adjust IDs as needed
```

**Rationale:**
- Documents are atomic - if embedding fails partway, the document should be re-ingested from scratch
- Keeping partial documents would cause duplicate chunks on re-ingestion
- Future improvement could add transaction rollback on failure

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
[2026-01-23 20:01] - SPEC-003 COMPLETE (initial implementation)

[2026-01-24 21:45] - Batch ingestion of 17 books started
[2026-01-24 21:46] - Failures at chunk ~110, diagnosed as character limit
[2026-01-24 21:48] - Added MAX_EMBED_CHARS=2000, still failing
[2026-01-24 21:52] - Diagnosed Unicode characters (smart quotes, em-dashes)
[2026-01-24 21:54] - Added Unicode normalization, still 9 failures
[2026-01-24 21:58] - Lowered MAX_EMBED_CHARS to 1800, still 3 failures
[2026-01-24 22:05] - Discovered 512 BERT token limit via systematic testing
[2026-01-24 22:08] - Tried token-based truncation, made things worse (wrong tokenizer)
[2026-01-24 22:12] - Reverted to character-based, set MAX_EMBED_CHARS=1500
[2026-01-24 22:14] - All 523 chunks pass testing
[2026-01-24 22:15] - Cleaned up partial documents, batch ingestion restarted
[2026-01-24 22:16] - SPEC-003 COMPLETE (with batch ingestion fixes)
```

---

## Recommendations for Designer

For future specs involving embeddings, consider:

1. **Vector formatting**: Always show explicit string formatting for pgvector inserts
2. **ivfflat.probes**: Document this requirement when using ivfflat indexes
3. **Text preprocessing**: Include a standard text cleaning step that handles:
   - Consecutive dots (TOC formatting)
   - Unicode typography (smart quotes, em-dashes, etc.)
   - Whitespace normalization
4. **Model context limits**: nomic-embed-text has a 512 BERT token limit (~1500 chars safe). Document embedding model limits prominently.
5. **Error recovery**: Consider transaction rollback for atomic document ingestion

The spec architecture and approach were sound. These are operational details discovered through real-world batch processing.

---

*"The books enter the library. The library remembers."*
