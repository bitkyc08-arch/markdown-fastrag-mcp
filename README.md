# MCP-Markdown-RAG

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-green)](LICENSE)
[![MCP Server](https://img.shields.io/badge/MCP-Server-blue)](https://modelcontextprotocol.io)
[![Python](https://img.shields.io/badge/Python-%3E%3D3.10-blue.svg)](https://python.org/)

A semantic search engine for your markdown documents. An MCP server that indexes notes, docs, and knowledge bases into a Milvus vector database, letting AI assistants find relevant content by **meaning**.

> Ask *"what are the tradeoffs of microservices?"* and find your notes about service boundaries, distributed systems, and API design — even if none of them mention "microservices."

## Why

AI assistants are limited by their context window. Instead of feeding entire document collections, this server lets them retrieve only the relevant sections:

- **Semantic matching** — finds conceptually related content, not just keyword hits
- **Multi-provider embeddings** — Gemini, OpenAI, Vertex AI, Voyage, or local models
- **Incremental indexing** — only re-indexes changed files by comparing hashes
- **MCP native** — works with any MCP host (Claude Code, Cursor, Windsurf, VS Code, etc.)

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
        "GEMINI_API_KEY": "${GEMINI_API_KEY}"
      }
    }
  }
}
```

## Embedding Providers

| Provider | `EMBEDDING_PROVIDER` | Required Env | Auth |
|----------|----------------------|--------------|------|
| **Gemini** | `gemini` | `GEMINI_API_KEY` | API key |
| **OpenAI** | `openai` | `OPENAI_API_KEY` | API key |
| **OpenAI-compatible** | `openai-compatible` | `EMBEDDING_API_KEY`, `EMBEDDING_BASE_URL` | API key |
| **Vertex AI** | `vertex` | `GOOGLE_APPLICATION_CREDENTIALS`, `VERTEX_PROJECT` | OAuth (auto-refresh) |
| **Voyage** | `voyage` | `VOYAGE_API_KEY` | API key |
| **Local** | `local` | — | — |

## Tools

| Tool | Description |
|------|-------------|
| `index_documents` | Index markdown files. Supports recursive directory scanning and incremental updates. |
| `search_documents` | Semantic search across indexed documents. Returns top-k results with relevance scores. |
| `clear_index` | Reset the vector database and tracking state. |

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `EMBEDDING_PROVIDER` | `local` | Provider: `gemini`, `openai`, `openai-compatible`, `vertex`, `voyage`, `local` |
| `EMBEDDING_MODEL` | (provider default) | Model name override |
| `EMBEDDING_DIM` | `768` | Vector dimension |
| `MILVUS_ADDRESS` | `.db/milvus_markdown.db` | Milvus server address or local file path |
| `GEMINI_API_KEY` | — | Gemini API key |
| `OPENAI_API_KEY` | — | OpenAI API key |
| `VOYAGE_API_KEY` | — | Voyage API key |
| `EMBEDDING_API_KEY` | — | OpenAI-compatible API key |
| `EMBEDDING_BASE_URL` | — | OpenAI-compatible base URL |
| `GOOGLE_APPLICATION_CREDENTIALS` | — | Service account JSON for Vertex |
| `VERTEX_PROJECT` | (auto) | GCP project ID |
| `VERTEX_LOCATION` | `us-central1` | Vertex AI region |

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

*Built on [MCP-Markdown-RAG](https://github.com/Zackriya-Solutions/MCP-Markdown-RAG) by Zackriya Solutions. Extended with multi-provider embeddings, configurable Milvus, batch processing, and incremental indexing.*
