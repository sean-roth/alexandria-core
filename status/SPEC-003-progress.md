# SPEC-003 Progress

**Spec:** [SPEC-003-book-ingestion.md](../specs/SPEC-003-book-ingestion.md)
**Started:** 2026-01-23
**Last Updated:** 2026-01-23
**Status:** COMPLETE

---

## Dependencies
- [x] PyMuPDF 1.26.7 installed
- [x] tiktoken 0.12.0 installed

## Book Storage
- [x] `/opt/alexandria/books/` directory created

## Ingestion Service
- [x] `/opt/alexandria/src/ingest.py` created
- [x] `/opt/alexandria/src/ingest_cli.py` created
- Notes:
  - Fixed vector formatting (spec bug): must format as string `"[0.1,...]"` for psycopg2
  - Fixed ivfflat.probes (spec bug): must SET before queries for small tables
  - Added text cleaning: consecutive dots (TOC formatting) cause Ollama 500 errors

## Validation
- [x] Test 1: Single book ingestion works
- [x] Test 2: Semantic search returns relevant chunks
- [x] Test 3: List command shows documents
- [x] Test 4: Database has correct data (768-dim embeddings)

## Blockers
- None

## Questions for Designer/Captain
- None - implementation complete

---

## Engineering Notes

### Bug Fix: Text Cleaning for Embeddings

Discovered that nomic-embed-text via Ollama fails with HTTP 500 when text contains
many consecutive dots (common in table of contents formatting like `........`).

**Solution:** Added `_clean_text_for_embedding()` method that:
- Collapses runs of 3+ dots to single ellipsis
- Normalizes whitespace

Raw text is preserved in database for retrieval; cleaning only applies to embedding generation.

### Spec Code Corrections Applied

1. **Vector formatting**: Spec passed raw Python lists to psycopg2. Fixed to format as
   pgvector string literals.

2. **ivfflat.probes**: Spec's `search_books()` missing probe configuration. Added
   `SET ivfflat.probes = 100` before search queries.

---

## Test Results

### Ingested Test Document
```
ID   Title                                              Chunks   Date
--------------------------------------------------------------------------------
2    WiFi Quick Start Guide (Test)                      4        2026-01-23
```

### Search Test
Query: "wifi connection linux"
- Top result distance: 0.3669
- Returns relevant chunks about Wi-Fi connection on Linux

### Database Verification
```
document_id | chunks | avg_dims
          2 |      4 |    768.0
```

---

## Session Log

```
[2026-01-23] - Dependencies installed (PyMuPDF, tiktoken)
[2026-01-23] - Books directory created
[2026-01-23] - ingest.py created with spec bug fixes
[2026-01-23] - ingest_cli.py created
[2026-01-23] - Discovered Ollama issue with consecutive dots
[2026-01-23] - Added text cleaning for embeddings
[2026-01-23] - All validation tests passed
[2026-01-23] - SPEC-003 COMPLETE
```
