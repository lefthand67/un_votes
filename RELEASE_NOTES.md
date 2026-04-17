# Release Notes

## v2.0.0 — 2026-04-11

Full rebuild of the UN Votes Database from the UN Digital Library.

### Database Statistics

| Metric | Value |
|---|---|
| Years | 1946–2026 |
| Resolutions | 10,110 |
| Resolutions (GA) | 7,370 |
| Resolutions (SC) | 2,740 |
| Recorded (with votes) | 8,439 |
| Non-recorded | 1,671 |
| Countries | 241 |
| Votes | 988,659 |

### Per-Decade Breakdown

| Decade | Total | GA | SC |
|---|---|---|---|
| 1940s | 330 | 256 | 74 |
| 1950s | 938 | 887 | 51 |
| 1960s | 829 | 692 | 137 |
| 1970s | 1,284 | 1,110 | 174 |
| 1980s | 1,643 | 1,458 | 185 |
| 1990s | 1,332 | 730 | 602 |
| 2000s | 1,377 | 762 | 615 |
| 2010s | 1,409 | 819 | 590 |
| 2020s | 968 | 656 | 312 |

### Data Integrity

- Zero lost records (all transactions committed successfully)
- Zero referential integrity violations across all FK checks
- All 81 years present (1946–2026), no gaps
- Spider exit status: `finished` (clean)

### What Changed from v1.x

- **Schema**: `readme_en` + `readme_ru` tables merged into single `readme` table with `lang` column (13 → 12 tables)
- **Spider**: rewritten with two-phase crawl architecture (Phase 1: collect IDs from listings, Phase 2: fetch details for new IDs only) to prevent silent data loss on HTTP 429 rate limiting
- **Middleware**: new `RateLimitBackoffMiddleware` with exponential backoff (30s → 300s, up to 8 retries) for HTTP 429 responses
- **Pipeline**: added data quality validation, per-record vote count tracking, integrity report on close
- **Post-crawl DDL**: `year_b` and `month_b` B-Tree indexes + `get_database_statistics()` function created automatically after crawl
- **Statistics function**: redesigned as bilingual (`'en'`/`'ru'`), transposed key-value format, GA/SC split, with `SET search_path = un` for caller independence
- **Logging**: all `print()` replaced with `logging.getLogger(__name__)`, dedicated `db_pipeline.log` file handler
- **Exit code**: `sys.exit(1)` on any lost records for CI/CD detection
- **User-Agent**: fixed to `python-requests/2.31.0` (browser UAs cause HTTP 202 async queue on UN Digital Library)

