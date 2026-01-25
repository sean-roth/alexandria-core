# SPEC-003a Progress

**Spec:** [SPEC-003a-duplicate-detection.md](../specs/SPEC-003a-duplicate-detection.md)
**Started:** 2026-01-25
**Last Updated:** 2026-01-25
**Status:** COMPLETE

---

## Implementation Summary

Added duplicate detection to the book ingestion pipeline to prevent re-ingesting books when running batch operations. This enables Sean's workflow of keeping all books in one folder with automated ingestion.

## Detection Method
- [x] `is_already_ingested()` method added to `BookIngestionService`
- [x] Checks by filename (not full path) in `library.documents`
- [x] Queries both `metadata->>'filename'` and `source_path LIKE` pattern

**Rationale:** Filename-based detection is simple and stable. Paths change (Windows → Linux transfers, folder reorganization), but filenames remain consistent identifiers. Content hashing would be more robust but overkill for current needs.

## Metadata Storage
- [x] `filename` now stored in document metadata during ingestion
- [x] Backfill migration for existing 18 documents

## ingest_book() Updates
- [x] Added `force: bool = False` parameter
- [x] Duplicate check runs before any processing
- [x] Returns `{"skipped": True, "reason": "already_ingested"}` when duplicate detected
- [x] Returns `{"skipped": False}` on successful ingestion

**Rationale:** `force` defaults to False for safety - accidental re-runs won't create duplicates. When `force=True`, duplicates are intentional (e.g., after changing chunking algorithm).

## CLI Updates
- [x] `--force` flag added to `ingest` command
- [x] `--force` flag added to `batch` command
- [x] `backfill` command added
- [x] Batch now reports: Ingested / Skipped / Failed counts

## Files Modified

| File | Changes |
|------|---------|
| `/opt/alexandria/src/ingest.py` | +`is_already_ingested()`, +`backfill_filenames()`, updated `ingest_book()` |
| `/opt/alexandria/src/ingest_cli.py` | +`--force` flags, +`backfill` command, batch counters |

---

## Validation

### Test 1: Backfill Existing Documents
```
$ python src/ingest_cli.py backfill
Updated 18 documents with filename metadata
```

Database verification:
```
title                                    | filename
-----------------------------------------+---------------------------------------------
aiengineering                            | aiengineering.pdf
architectingdataandmachinelearningplatfo | architectingdataandmachinelearningplatforms.pdf
generativeaionaws                        | generativeaionaws.pdf
```

### Test 2: Duplicate Detection
```
$ python src/ingest_cli.py ingest /opt/alexandria/books/aiengineering.pdf
Skipping (already ingested): aiengineering.pdf

Skipped (already ingested)
```

### Test 3: Batch with Mixed Content
Created test directory with 1 ingested book + 1 new book:
```
$ python src/ingest_cli.py batch /tmp/alexandria-test/
Found 2 PDF files

Skipping (already ingested): aiengineering.pdf
Ingesting: testbook
  Extracting text...
  Found 24 pages with text
  Chunking...
  Created 8 chunks
  ...
  -> 8 chunks

Batch complete:
  Ingested: 1
  Skipped:  1
  Failed:   0
```

---

## Design Decisions

### Why filename, not content hash?
- Simpler implementation
- Filenames are already unique in practice (book titles)
- No additional processing overhead
- Can add content hashing later if needed

### Why check both metadata and source_path?
- Backward compatibility: existing documents only had `source_path`
- After backfill, both fields contain filename
- Belt-and-suspenders approach for reliability

### Why return dict with skipped flag vs raising exception?
- Skipping is expected behavior, not an error
- Allows batch operations to continue smoothly
- Caller can easily check `result.get("skipped")` to branch logic

---

## Blockers
- None

## Questions for Designer/Captain
- None - implementation complete per spec

---

## Session Log

```
[2026-01-25] - Pulled SPEC-003a from GitHub
[2026-01-25] - Planned implementation (entered plan mode)
[2026-01-25] - Added is_already_ingested() method
[2026-01-25] - Added backfill_filenames() method
[2026-01-25] - Updated ingest_book() with force parameter and skip logic
[2026-01-25] - Updated CLI with --force flags and backfill command
[2026-01-25] - Ran backfill: 18 documents updated
[2026-01-25] - All validation tests passed
[2026-01-25] - SPEC-003a COMPLETE
```
