# Phase 3: Search Result Format Enhancement

**Date**: 2026-02-15

## Problem

`search_documents` returned only `filename` + `text` snippet.
Two useful fields were available from Milvus but discarded:

- `distance` — cosine similarity score (0–1)
- `path` — full file path relative to workspace

This made it hard to judge result relevance or locate files in the vault.

## Change

**File**: `server.py` line 373–378 (search result formatting)

**Before**:
```python
f"File: **{res.entity.filename}**\n---\nText: {res.entity.text}\n---\n"
```

**After**:
```python
f"File: **{res.entity.filename}** (relevance: {res.distance:.1%})\n"
f"Path: `{res.entity.path}`\n---\n"
f"Text: {res.entity.text}\n---\n"
```

## Impact

- Zero schema change — `path` and `distance` were already stored in Milvus and parsed into `SearchResult`
- No reindexing required
- Server restart required to pick up code change

## Example Output (before → after)

**Before**:
```
File: **일일-루틴-정리.md**
Text: # 일일 루틴 정리
```

**After**:
```
File: **일일-루틴-정리.md** (relevance: 72.3%)
Path: `300_permanent/health/일일-루틴-정리.md`
Text: # 일일 루틴 정리
```

## Future Improvements

- Add line/chunk number tracking (requires indexing schema change)
- Return structured JSON instead of markdown string for programmatic use
