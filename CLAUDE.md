# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Shopkeeper Agent** (掌柜问数) — enterprise NL2SQL system that converts natural language questions into SQL queries against a data warehouse. Uses hybrid retrieval (vector + full-text) + multi-stage reasoning to generate accurate SQL, rather than relying solely on LLM output.

## Quick Commands

```bash
# Install dependencies
uv sync
uv sync --dev          # include dev deps (ruff, pre-commit)

# Start local services (MySQL, Qdrant, ES, Kibana, TEI Embedding)
docker compose -f docker/docker-compose.yaml up -d

# Lint / format
ruff check .
ruff format .

# Build metadata knowledge base (tables → Qdrant vectors, values → ES index)
uv run python -m app.scripts.build_meta_knowledge.py -c conf/meta_config.yaml

# Test LLM config
uv run python -m app.agent.llm

# Run the agent locally (has built-in test in graph.py)
uv run python -m app.agent.graph
```

## Architecture

Two main pipelines:

1. **Knowledge Build Pipeline**: `conf/meta_config.yaml` → `MetaKnowledgeService` → MySQL (structured metadata) + Qdrant (vector embeddings) + Elasticsearch (full-text value index)
2. **Query Pipeline**: User question → LangGraph agent → SQL result

### LangGraph Agent Workflow (`app/agent/graph.py`)

```
START → extract_keywords → [recall_column, recall_value, recall_metric] (parallel)
    → merge_retrieved_info → [filter_table, filter_metric] (parallel)
    → add_extra_context → generate_sql → validate_sql
    → (error?) correct_sql → run_sql → END
    → (no error) run_sql → END
```

### Layer Structure

| Layer | Path | Responsibility |
|-------|------|---------------|
| Config | `conf/*.yaml` → `app/conf/` | OmegaConf + dataclass, YAML with `${oc.env:VAR}` interpolation from `.env` |
| Clients | `app/clients/` | Singleton managers for Qdrant, ES, MySQL (meta + dw), Embedding — `.init()` + `.close()` lifecycle |
| Repositories | `app/repositories/` | Data access: `es/` for value full-text, `qdrant/` for column/metric vectors, `mysql/meta/` for metadata ORM, `mysql/dw/` for data warehouse queries |
| Entities | `app/entities/` | Business objects (ColumnInfo, MetricInfo, ValueInfo, TableInfo) |
| Models | `app/models/` | SQLAlchemy ORM models |
| Services | `app/services/` | Business orchestration (currently: `MetaKnowledgeService`) |
| Agent | `app/agent/` | LangGraph graph, state, context, LLM init, node functions |
| Prompts | `prompts/*.prompt` | Template files loaded by `app/prompt/prompt_loader.py` via `load_prompt("name")` |

### State vs Context (`app/agent/state.py`, `app/agent/context.py`)

- **State** (`DataAgentState`): LangGraph node data — `query`, `keywords`, `retrieved_column_infos`, `retrieved_metric_infos`, `retrieved_value_infos`, `error`. Nodes return partial dicts to update state.
- **Context** (`DataAgentContext`): Runtime dependencies not part of state — repositories and embedding client, accessed via `runtime.context["key"]`.

### Node Conventions (`app/agent/nodes/`)

Every node is `async def node_name(state, runtime)`:
- Read from `state` dict for business data
- Read from `runtime.context` for external tools
- Write progress via `runtime.stream_writer`
- Return `dict` with keys matching `DataAgentState` fields to update state

### Prompt Templates

Files in `prompts/` directory (plain `.prompt` extension), loaded by name:
```python
from app.prompt.prompt_loader import load_prompt
prompt_text = load_prompt("extend_keywords_for_column_recall")
```

## Infrastructure

Docker Compose services (all ports in `conf/app_config.yaml`):
- MySQL: 3306 (databases: `meta` for metadata, `dw` for data warehouse)
- Qdrant: 6333
- Elasticsearch: 9200
- Kibana: 5601
- TEI Embedding (BAAI/bge-large-zh-v1.5): 8081

## Configuration

Environment variable `LLM_API_KEY` required in `.env` (copy from `.env.example`). The LLM defaults to SiliconFlow's OpenAI-compatible endpoint with model `Pro/zai-org/GLM-5.1`.

## Python Version

Requires Python >= 3.14 (specified in `pyproject.toml`).
