# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# From the project root:
./run.sh

# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app serves at `http://localhost:8000` (web UI) and `http://localhost:8000/docs` (FastAPI OpenAPI docs).

**Setup**: Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`.

Install dependencies with `uv sync` (requires Python 3.13+). Always use `uv run python <file>` to run Python files.

## Architecture

The system is a RAG chatbot where **Claude drives retrieval** via tool use rather than a traditional retrieve-then-generate pipeline.

**Request flow:**
1. Frontend (`frontend/`) sends `POST /api/query` → `backend/app.py`
2. `RAGSystem.query()` (`rag_system.py`) passes the query to `AIGenerator`
3. Claude decides whether to call the `search_course_content` tool (max one call per query)
4. If tool called: `CourseSearchTool` executes a semantic search against ChromaDB and returns formatted snippets
5. Claude synthesizes the final answer; sources and session history are updated

**Component responsibilities:**
- `rag_system.py` — orchestrator; wires all components together and is the single entry point for queries
- `ai_generator.py` — all Claude API calls, including the tool-use agentic loop (`generate_response` + `_handle_tool_execution`)
- `vector_store.py` — ChromaDB wrapper with two collections: `course_catalog` (course titles/metadata for fuzzy name resolution) and `course_content` (chunked lesson text for semantic search)
- `document_processor.py` — parses the custom `.txt` course format into `Course`/`Lesson`/`CourseChunk` objects; also handles chunking
- `search_tools.py` — `CourseSearchTool` (the Anthropic tool definition + execution) and `ToolManager` (registry)
- `session_manager.py` — in-memory conversation history per session (lost on restart; last `MAX_HISTORY=2` exchanges kept)
- `models.py` — Pydantic models: `Course`, `Lesson`, `CourseChunk`
- `config.py` — all tuneable parameters (model name, chunk size, ChromaDB path, etc.)

## Course Document Format

Course `.txt` files in `docs/` must follow this structure (parsed by `document_processor.py`):

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<lesson content...>

Lesson 1: <lesson title>
Lesson Link: <url>
<lesson content...>
```

Documents are auto-loaded from `../docs` on server startup. Duplicate courses (matched by title) are skipped.

## Key Configuration (`backend/config.py`)

| Setting | Default | Notes |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Change here to upgrade model |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Local sentence-transformers model |
| `CHUNK_SIZE` | 800 | Characters per chunk |
| `CHUNK_OVERLAP` | 100 | Overlap between chunks |
| `MAX_RESULTS` | 5 | Search results returned to Claude |
| `MAX_HISTORY` | 2 | Conversation exchanges kept per session |
| `CHROMA_PATH` | `./chroma_db` | Persistent vector DB location (relative to `backend/`) |

## Adding a New Tool

1. Create a class inheriting from `Tool` (ABC in `search_tools.py`) and implement `get_tool_definition()` and `execute()`
2. Register it in `RAGSystem.__init__()` via `self.tool_manager.register_tool(...)`

The `ToolManager` handles multi-tool dispatch; `AIGenerator._handle_tool_execution` loops through all tool calls in a single response and feeds results back to Claude.
