# Phase 2: Search Quality & Config Improvements

**날짜**: 2026-02-15
**목표**: 메타 파일 제외, MILVUS_INSERT_BATCH 배치 처리, CHUNK_SIZE 복원

---

## 배경

1. `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` 같은 에이전트 설정 파일이 마크다운 RAG에 인덱싱되어 검색 노이즈 발생
2. `MARKDOWN_CHUNK_SIZE`가 5000으로 잘못 설정 (Milvus insert batch와 혼동)
3. `server.py`의 `index_documents()`가 전체 데이터를 단일 Milvus insert로 전송 → gRPC 64MB 초과 가능

---

## 변경 사항

### 1. 메타 파일 제외 (`utils.py`)

`list_md_files()` 함수에 파일 레벨 필터링 추가:

```python
_DEFAULT_EXCLUDE_FILES = {"AGENTS.md", "CLAUDE.md", "GEMINI.md"}
_extra_files = os.getenv("MARKDOWN_EXCLUDE_FILES", "")
EXCLUDE_FILES = _DEFAULT_EXCLUDE_FILES | (
    set(_extra_files.split(",")) if _extra_files else set()
)
```

- 기본: `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` 제외
- `MARKDOWN_EXCLUDE_FILES` env로 추가 파일 지정 가능 (쉼표 구분)
- `list_md_files()` 내 `os.walk()` 루프에서 `basename in EXCLUDE_FILES` 체크

### 2. `MARKDOWN_CHUNK_SIZE` 복원 (4개 config)

**문제**: 이전에 `MARKDOWN_CHUNK_SIZE`를 5000으로 변경했으나, 이 값은 **텍스트 청킹 크기**이지 Milvus insert 배치 크기가 아님. 큰 청크 = 검색 정밀도 하락.

**수정**: 4개 에이전트 MCP 설정 전부 `2048`로 복원

| config 파일                                        | Before | After    |
| -------------------------------------------------- | ------ | -------- |
| `~/.gemini/antigravity/mcp_config.json`            | 5000   | **2048** |
| `~/.codex/config.toml`                             | 5000   | **2048** |
| `~/Library/Application Support/Code/User/mcp.json` | 5000   | **2048** |
| `~/.claude/settings.local.json`                    | 5000   | **2048** |

### 3. `MILVUS_INSERT_BATCH` 설정 추가

4개 MCP config에 `MILVUS_INSERT_BATCH=5000` 추가:

```json
"MILVUS_INSERT_BATCH": "5000"
```

이 값은 `reindex.py`와 `server.py` 양쪽에서 사용.

### 4. `server.py` 배치 insert 적용

**수정 전**: 전체 데이터를 한 번에 insert (23K 청크 × 768d vector = 92MB → gRPC 64MB 초과)

```python
res = milvus_client.insert(collection_name=COLLECTION_NAME, data=data)
```

**수정 후**: env 기반 배치 insert

```python
MILVUS_INSERT_BATCH = int(os.getenv("MILVUS_INSERT_BATCH", "5000"))
res = {}
for i in range(0, len(data), MILVUS_INSERT_BATCH):
    batch = data[i : i + MILVUS_INSERT_BATCH]
    res = milvus_client.insert(collection_name=COLLECTION_NAME, data=batch)
```

---

## 최종 MCP 설정 (markdown-rag)

| 키                         | 값         | 용도                                |
| -------------------------- | ---------- | ----------------------------------- |
| `MARKDOWN_CHUNK_SIZE`      | 2048       | 텍스트 청킹 크기                    |
| `EMBEDDING_BATCH_SIZE`     | 100        | Vertex AI 임베딩 배치               |
| `EMBEDDING_BATCH_DELAY_MS` | 1000       | 429 방지 딜레이                     |
| `MILVUS_INSERT_BATCH`      | 5000       | Milvus insert 배치 (gRPC 64MB 방지) |
| `MARKDOWN_EXCLUDE_FILES`   | (optional) | 추가 제외 파일                      |

---

## 수정 파일 목록

| 파일           | 변경 내용                                     |
| -------------- | --------------------------------------------- |
| `utils.py`     | 메타 파일 제외 (`EXCLUDE_FILES`)              |
| `server.py`    | 배치 insert (`MILVUS_INSERT_BATCH` env)       |
| 4개 MCP config | `CHUNK_SIZE=2048`, `MILVUS_INSERT_BATCH=5000` |
