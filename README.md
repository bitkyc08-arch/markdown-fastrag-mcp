# MCP-Markdown-RAG

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-green)](LICENSE)
[![MCP Server](https://img.shields.io/badge/MCP-Server-blue)](https://modelcontextprotocol.io)
[![Python](https://img.shields.io/badge/Python-%3E%3D3.10-blue.svg)](https://python.org/)

A semantic search engine for your markdown documents. An MCP server that indexes notes, docs, and knowledge bases into a Milvus vector database, letting AI assistants find relevant content by **meaning**.

> Ask *"what are the tradeoffs of microservices?"* and find your notes about service boundaries, distributed systems, and API design — even if none of them mention "microservices."

## Features

- **Semantic matching** — finds conceptually related content, not just keyword hits
- **Multi-provider embeddings** — Gemini, OpenAI, Vertex AI, Voyage, or local models
- **Smart incremental indexing** — mtime/size fast-path skips unchanged files without reading them; hash only computed when metadata changes
- **Single-pass delta scan** — detects new, changed, and deleted files in one directory walk
- **Stale vector pruning** — automatically removes vectors for deleted or moved files from Milvus
- **Batch embedding** — concurrent batches with rate-limit retry (429 exponential backoff)
- **Batch insert** — chunked Milvus inserts to stay under the gRPC 64MB message limit
- **Shell reindex CLI** — `reindex.py` for large-scale indexing with real-time progress logs
- **Configurable exclusions** — skip directories (`node_modules`, `.git`, `_legacy`) and files (`AGENTS.md`) via env
- **Milvus Standalone support** — connect to a Docker-based Milvus server for multi-agent concurrent access
- **MCP native** — works with any MCP host (Claude Code, Cursor, Windsurf, VS Code, Antigravity, Codex, etc.)

## Quick Start

Requires [uv](https://docs.astral.sh/uv/) (Python package manager).

### 1. Clone

```bash
git clone https://github.com/bitkyc08-arch/mcp-markdown-rag.git
```

### 2. Configure

Add to your MCP host config:

```json
{
  "mcpServers": {
    "markdown-rag": {
      "command": "uv",
      "args": [
        "--directory", "/path/to/mcp-markdown-rag",
        "run", "server.py"
      ],
      "env": {
        "EMBEDDING_PROVIDER": "gemini",
        "EMBEDDING_MODEL": "gemini-embedding-001",
        "EMBEDDING_DIM": "768",
        "GEMINI_API_KEY": "${GEMINI_API_KEY}",
        "MILVUS_ADDRESS": "http://localhost:19530"
      }
    }
  }
}
```

> **Tip**: For local-only use (no Docker), omit `MILVUS_ADDRESS` — it defaults to a local SQLite-based Milvus Lite file (`.db/milvus_markdown.db`).

## Embedding Providers

| Provider              | `EMBEDDING_PROVIDER` | Required Env                                       | Auth                           |
| --------------------- | -------------------- | -------------------------------------------------- | ------------------------------ |
| **Gemini**            | `gemini`             | `GEMINI_API_KEY`                                   | API key                        |
| **OpenAI**            | `openai`             | `OPENAI_API_KEY`                                   | API key                        |
| **OpenAI-compatible** | `openai-compatible`  | `EMBEDDING_API_KEY`, `EMBEDDING_BASE_URL`          | API key                        |
| **Vertex AI**         | `vertex`             | `GOOGLE_APPLICATION_CREDENTIALS`, `VERTEX_PROJECT` | Service Account (auto-refresh) |
| **Voyage**            | `voyage`             | `VOYAGE_API_KEY`                                   | API key                        |
| **Local**             | `local`              | —                                                  | —                              |

## Tools

| Tool               | Description                                                                                                                             |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| `index_documents`  | Index markdown files with incremental updates. Automatically detects new, changed, and deleted files. Prunes stale vectors from Milvus. |
| `search_documents` | Semantic search across indexed documents. Returns top-k results with relevance scores and file paths.                                   |
| `clear_index`      | Reset the vector database and tracking state.                                                                                           |

## Incremental Indexing & Pruning

The indexing engine uses a **single-pass delta scan** to efficiently detect what changed:

```
Directory walk (1 pass)
    ├── New file?           → index
    ├── mtime/size same?    → skip (no file read, no hash)
    ├── mtime/size changed? → compute hash
    │   ├── hash different? → re-index
    │   └── hash same?      → update tracking only (e.g. `touch`)
    └── Tracked but missing? → prune from Milvus + remove from tracking
```

**Performance** (1300+ files, 1 file changed):

| Metric                              | Result                              |
| ----------------------------------- | ----------------------------------- |
| Unchanged files — hash computations | **0** (mtime/size fast-path)        |
| Changed file — embed + insert       | **~3 seconds**                      |
| No changes — full scan              | **instant** ("Already up to date!") |
| Deleted file — prune + scan         | **instant**                         |

### How pruning works

When a file is deleted or moved to an excluded directory (e.g. `_legacy/`), the next incremental `index_documents` call will:

1. Detect the file is missing from the current directory scan
2. Delete its vectors from Milvus (`filter: path == '...'`)
3. Remove it from the tracking file
4. Return a message: `"Pruned N deleted/moved files."`

No manual cleanup needed — just delete the file and re-index.

## Shell Reindex CLI

For large-scale indexing (1000+ files), use `reindex.py` directly for real-time logs and better error handling:

```bash
cd /path/to/mcp-markdown-rag

# Incremental (changed files only)
EMBEDDING_PROVIDER=vertex \
MILVUS_ADDRESS=http://localhost:19530 \
GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa.json \
VERTEX_PROJECT=your-project-id \
VERTEX_LOCATION=us-central1 \
uv run python reindex.py /path/to/vault

# Full rebuild (drop + re-create collection)
uv run python reindex.py /path/to/vault --force
```

Features over MCP `index_documents`:
- Real-time progress logs (batch N/M, elapsed time)
- 429 rate-limit retry with exponential backoff (5 attempts)
- Chunked Milvus insert (configurable via `MILVUS_INSERT_BATCH`)
- Non-recursive mode (`--no-recursive`)

## Configuration

### Core

| Variable             | Default                  | Description                                                          |
| -------------------- | ------------------------ | -------------------------------------------------------------------- |
| `EMBEDDING_PROVIDER` | `local`                  | `gemini`, `openai`, `openai-compatible`, `vertex`, `voyage`, `local` |
| `EMBEDDING_MODEL`    | (provider default)       | Model name override                                                  |
| `EMBEDDING_DIM`      | `768`                    | Vector dimension                                                     |
| `MILVUS_ADDRESS`     | `.db/milvus_markdown.db` | Milvus address (`http://host:port`) or local file path               |

### Indexing Tuning

| Variable                       | Default | Description                                                                |
| ------------------------------ | ------- | -------------------------------------------------------------------------- |
| `MARKDOWN_CHUNK_SIZE`          | `2048`  | Token chunk size for splitting documents                                   |
| `MARKDOWN_CHUNK_OVERLAP`       | `100`   | Token overlap between chunks                                               |
| `EMBEDDING_BATCH_SIZE`         | `250`   | Texts per embedding API call                                               |
| `EMBEDDING_BATCH_DELAY_MS`     | `0`     | Delay between embedding batches (ms). Set to `1000` for rate-limited APIs. |
| `EMBEDDING_CONCURRENT_BATCHES` | `4`     | Parallel embedding batches                                                 |
| `MILVUS_INSERT_BATCH`          | `5000`  | Rows per Milvus insert call (gRPC 64MB limit)                              |

### Exclusions

| Variable                 | Default | Description                                                                                                                                    |
| ------------------------ | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `MARKDOWN_EXCLUDE_DIRS`  | —       | Extra directories to exclude (comma-separated). Added to built-in: `node_modules`, `__pycache__`, `devlog`, `_legacy`, `dist`, `build`, `.git` |
| `MARKDOWN_EXCLUDE_FILES` | —       | Extra files to exclude (comma-separated). Added to built-in: `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`                                             |

### Provider Auth

| Variable                         | Description                                 |
| -------------------------------- | ------------------------------------------- |
| `GEMINI_API_KEY`                 | Gemini API key                              |
| `OPENAI_API_KEY`                 | OpenAI API key                              |
| `VOYAGE_API_KEY`                 | Voyage API key                              |
| `EMBEDDING_API_KEY`              | OpenAI-compatible API key                   |
| `EMBEDDING_BASE_URL`             | OpenAI-compatible base URL                  |
| `GOOGLE_APPLICATION_CREDENTIALS` | Service account JSON path for Vertex AI     |
| `VERTEX_PROJECT`                 | GCP project ID (auto-detected if SA has it) |
| `VERTEX_LOCATION`                | Vertex AI region (default: `us-central1`)   |

### Vertex AI Example

```json
{
  "mcpServers": {
    "markdown-rag": {
      "command": "uv",
      "args": ["--directory", "/path/to/mcp-markdown-rag", "run", "server.py"],
      "env": {
        "EMBEDDING_PROVIDER": "vertex",
        "EMBEDDING_MODEL": "gemini-embedding-001",
        "EMBEDDING_DIM": "768",
        "MARKDOWN_CHUNK_SIZE": "2048",
        "MARKDOWN_CHUNK_OVERLAP": "120",
        "EMBEDDING_BATCH_SIZE": "100",
        "EMBEDDING_BATCH_DELAY_MS": "1000",
        "EMBEDDING_CONCURRENT_BATCHES": "3",
        "MILVUS_INSERT_BATCH": "5000",
        "MILVUS_ADDRESS": "http://localhost:19530",
        "GOOGLE_APPLICATION_CREDENTIALS": "/path/to/service-account.json",
        "VERTEX_PROJECT": "your-gcp-project-id",
        "VERTEX_LOCATION": "us-central1"
      }
    }
  }
}
```

## Debugging

```bash
npx @modelcontextprotocol/inspector uv --directory /path/to/mcp-markdown-rag run server.py
```

## License

Apache License 2.0 — see [LICENSE](LICENSE).

---

*Forked from [MCP-Markdown-RAG](https://github.com/Zackriya-Solutions/MCP-Markdown-RAG) by Zackriya Solutions. Extended with multi-provider embeddings (Vertex AI native), single-pass incremental indexing with mtime/size fast-path, stale vector pruning, batch processing, shell reindex CLI, configurable exclusions, and Milvus Standalone support.*
