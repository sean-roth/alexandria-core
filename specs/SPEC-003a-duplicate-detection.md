# SPEC-003a: Duplicate Detection for Book Ingestion

**Project:** alexandria-core  
**Author:** Claude (Designer) via claude.ai  
**For:** Claude Code (Engineer) on Linux Server  
**Captain:** Sean  
**Date:** January 24, 2026  
**Status:** Ready for Implementation  
**Depends on:** SPEC-003 (complete)  
**Type:** Enhancement Patch

---

## Context

SPEC-003 built book ingestion but has no duplicate detection. If you run batch ingestion twice on the same folder, every book gets ingested again - creating duplicate entries, duplicate chunks, and duplicate search results.

Sean wants all books in one folder with automated ingestion. This requires the system to recognize what's already been processed.

---

## This Spec's Goal

Add duplicate detection to the ingestion pipeline:
1. Check if a book has already been ingested before processing
2. Skip duplicates by default
3. Provide override option when re-ingestion is intentional
4. Report what was skipped

---

## Requirement 1: Detection Method

Check by **filename** (not full path) in `library.documents`.

```python
def is_already_ingested(self, pdf_path: str) -> bool:
    """Check if this book has already been ingested."""
    filename = os.path.basename(pdf_path)
    
    conn = psycopg2.connect(**self.db_config)
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT id, title FROM library.documents
        WHERE metadata->>'filename' = %s
        OR source_path LIKE %s
    """, (filename, f'%{filename}'))
    
    result = cursor.fetchone()
    conn.close()
    
    return result is not None
```

**Why filename, not full path?**
- Paths change (Windows → Linux transfer)
- Same book in different folders is still the same book
- Filename is stable identifier

---

## Requirement 2: Store Filename in Metadata

Update `ingest_book()` to always store filename:

```python
doc_metadata = metadata or {}
doc_metadata["source_path"] = pdf_path
doc_metadata["filename"] = os.path.basename(pdf_path)  # ADD THIS
doc_metadata["page_count"] = len(pages)
doc_metadata["chunk_count"] = len(chunks)
```

---

## Requirement 3: Update ingest_book()

Add skip logic:

```python
def ingest_book(
    self,
    pdf_path: str,
    title: Optional[str] = None,
    metadata: Optional[dict] = None,
    force: bool = False  # ADD THIS PARAMETER
) -> dict:
    """
    Ingest a PDF book into the library.
    
    Args:
        pdf_path: Path to PDF file
        title: Optional title (defaults to filename)
        metadata: Optional additional metadata
        force: If True, re-ingest even if already exists
    
    Returns: {"document_id": int, "chunks_created": int, "total_tokens": int, "skipped": bool}
    """
    if not os.path.exists(pdf_path):
        raise FileNotFoundError(f"PDF not found: {pdf_path}")
    
    # Check for duplicate
    if not force and self.is_already_ingested(pdf_path):
        filename = os.path.basename(pdf_path)
        print(f"Skipping (already ingested): {filename}")
        return {
            "document_id": None,
            "chunks_created": 0,
            "total_tokens": 0,
            "skipped": True,
            "reason": "already_ingested"
        }
    
    # ... rest of existing ingestion code ...
    
    return {
        "document_id": document_id,
        "chunks_created": len(chunks),
        "total_tokens": total_tokens,
        "skipped": False
    }
```

---

## Requirement 4: Update CLI

Update `ingest_cli.py` with new flags:

```python
# Ingest command
ingest_parser = subparsers.add_parser("ingest", help="Ingest a PDF book")
ingest_parser.add_argument("pdf_path", help="Path to PDF file")
ingest_parser.add_argument("--title", help="Book title (default: filename)")
ingest_parser.add_argument("--force", action="store_true", 
                           help="Re-ingest even if already exists")

# Batch command  
batch_parser = subparsers.add_parser("batch", help="Ingest all PDFs in a directory")
batch_parser.add_argument("directory", help="Directory containing PDFs")
batch_parser.add_argument("--force", action="store_true",
                          help="Re-ingest all, even if already exists")
```

Update batch handler to track and report:

```python
elif args.command == "batch":
    if not os.path.isdir(args.directory):
        print(f"Error: {args.directory} is not a directory")
        sys.exit(1)
    
    pdfs = [f for f in os.listdir(args.directory) if f.lower().endswith('.pdf')]
    print(f"Found {len(pdfs)} PDF files\n")
    
    ingested = 0
    skipped = 0
    failed = 0
    
    for pdf in sorted(pdfs):
        pdf_path = os.path.join(args.directory, pdf)
        try:
            result = service.ingest_book(pdf_path, force=args.force)
            if result.get("skipped"):
                skipped += 1
            else:
                ingested += 1
                print(f"  ✓ {result['chunks_created']} chunks\n")
        except Exception as e:
            failed += 1
            print(f"  ✗ Error: {e}\n")
    
    print(f"\nBatch complete:")
    print(f"  Ingested: {ingested}")
    print(f"  Skipped:  {skipped}")
    print(f"  Failed:   {failed}")
```

---

## Requirement 5: Backfill Existing Documents

The 17 already-ingested books don't have `filename` in metadata. Run this migration:

```sql
UPDATE library.documents
SET metadata = metadata || jsonb_build_object('filename', 
    substring(source_path from '[^/\\]+$'))
WHERE metadata->>'filename' IS NULL;
```

Or in Python (safer):

```python
def backfill_filenames(self):
    """Add filename to metadata for existing documents."""
    conn = psycopg2.connect(**self.db_config)
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT id, source_path FROM library.documents
        WHERE metadata->>'filename' IS NULL
    """)
    
    for row in cursor.fetchall():
        doc_id, source_path = row
        filename = os.path.basename(source_path)
        cursor.execute("""
            UPDATE library.documents
            SET metadata = metadata || %s
            WHERE id = %s
        """, (Json({"filename": filename}), doc_id))
    
    conn.commit()
    conn.close()
```

Add CLI command:

```python
# Backfill command
backfill_parser = subparsers.add_parser("backfill", help="Backfill filename metadata")
```

---

## Validation

### Test 1: Backfill Existing

```bash
python src/ingest_cli.py backfill
```

Verify:
```sql
SELECT title, metadata->>'filename' FROM library.documents;
```

### Test 2: Skip Duplicate

```bash
# Try to ingest an already-ingested book
python src/ingest_cli.py ingest /opt/alexandria/books/aiengineering.pdf
# Should print: Skipping (already ingested): aiengineering.pdf
```

### Test 3: Force Re-ingest

```bash
python src/ingest_cli.py ingest /opt/alexandria/books/aiengineering.pdf --force
# Should ingest again (creates duplicate - for testing only)
```

### Test 4: Batch with Mixed Content

```bash
# With some new and some existing books
python src/ingest_cli.py batch /opt/alexandria/books/
# Should show: Ingested: X, Skipped: Y, Failed: Z
```

---

## Success Criteria

- [ ] `is_already_ingested()` method added
- [ ] `ingest_book()` checks for duplicates
- [ ] `--force` flag works for single and batch
- [ ] Batch reports skipped count
- [ ] Backfill command works
- [ ] Existing 17 books have filename in metadata

---

## Notes for Claude Code

### On Detection Strategy

Using filename is simple and works. Future enhancement could hash file contents for true deduplication (same book, different filename), but that's overkill for now.

### On Force Flag

`--force` is for intentional re-ingestion (e.g., if chunking algorithm changes). It will create duplicates - that's expected. User responsibility.

### On Backfill

Run backfill once after implementing. New ingestions will automatically have filename in metadata.

---

*"The library knows what it holds."*
