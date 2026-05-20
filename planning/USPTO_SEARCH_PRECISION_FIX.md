# Fix: USPTO API Search Narrowed to Field-Specific Queries

**Status: COMPLETED** (2026-02-18)

## Context

The `search_by_assignee()` function returned irrelevant results (X-ray fluoroscopy, light-emitting devices) when searching for "Intel Corporation". Root cause: `_search_uspto_odp()` passed the company name as a generic keyword query (`q=Intel Corporation`), which the USPTO API treats as `Intel OR Corporation` across ALL fields — returning 1.9M results.

## Problem

**Before** (`search_by_assignee` line ~203):
```python
results = _search_uspto_odp(company, limit)  # "Intel Corporation" → matches ANY field
```

Result: `q=Intel+Corporation` → 1,958,699 results (keyword OR match across all fields)

## Fix Applied

### `search_by_assignee()` — uses nested `applicantNameText` field
```python
assignee_query = f'applicationMetaData.applicantBag.applicantNameText:"{company}"'
results = _search_uspto_odp(assignee_query, limit)
```

Result: `q=applicationMetaData.applicantBag.applicantNameText%3A%22Intel+Corporation%22` → 52,403 results (all Intel Corporation patents)

**Note:** The USPTO ODP API requires fully-qualified nested field paths (e.g., `applicationMetaData.applicantBag.applicantNameText`), not short-form names (e.g., `applicantNameText`). Short-form returns 404. This was discovered during implementation — the original plan assumed Lucene short-form syntax would work.

### `search_by_title()` — uses nested `inventionTitle` field
```python
title_query = f'applicationMetaData.inventionTitle:({keywords})'
results = _search_uspto_odp(title_query, limit)
```

### `_search_uspto_odp()` — unchanged
It just passes whatever query string it receives as the `q` parameter.

## Files Modified

- `tools/patent_search.py`
- `.env` — added `USPTO_API_KEY` (already gitignored)
- `planning/REPO_SETUP_PROGRESS.md`

## Verification Results

All tests passed against the live USPTO ODP API:

| Test | Result |
|------|--------|
| `search_by_assignee('Intel Corporation', limit=5)` | 52,403 total, all 5 returned show `Intel Corporation` as assignee |
| `search_by_assignee('Microsoft', limit=3)` | 54,797 total, returns `Microsoft Technology Licensing, LLC` patents |
| `search_by_assignee('Apple', limit=3)` | 56,326 total, returns `Apple Inc.` patents |
| `search_by_title('semiconductor', limit=3)` | 225,910 total, all titles contain "semiconductor" |

## Technical Details

### USPTO ODP Field Paths

The USPTO Open Data Portal uses Elasticsearch-style nested field queries. Common paths:

| Search Type | Field Path | Example |
|:------------|:-----------|:--------|
| Assignee/Company | `applicationMetaData.applicantBag.applicantNameText` | `Intel Corporation` |
| Invention Title | `applicationMetaData.inventionTitle` | `semiconductor` |
| Patent Number | `patentNumber` | `10123456` |
| Filing Date | `applicationMetaData.filingDate` | `2020-01-01` |
| CPC Code | `patentCPCText` | `H01L` |

### Query Syntax

- **Field search:** `field:"value"` → exact match
- **Field search:** `field:(term1 OR term2)` → multiple terms
- **Range search:** `field:[start TO end]` → range
- **Generic keyword:** `keyword` → searches all fields

### API Rate Limiting

- **Limit:** 500 requests/second (generous for a public API)
- **Timeout:** 30 seconds per query
- **Fallback:** If one query fails, `_search_uspto_odp()` retries with simpler syntax
