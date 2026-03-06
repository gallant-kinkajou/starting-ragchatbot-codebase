# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# Install dependencies (one-time)
uv sync

# Start the server

./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app is served at `http://localhost:8000`. The frontend is served as static files by FastAPI from the `frontend/` directory.

Requires a `.env` file in the project root:
```
AWS_PROFILE=your_aws_profile_here
```

## Architecture

This is a RAG (Retrieval-Augmented Generation) chatbot that answers questions about course materials.

**Stack:** FastAPI backend + vanilla JS/HTML frontend + ChromaDB vector store + Anthropic Claude (tool-use pattern)

### Query Flow

1. Frontend (`frontend/script.js`) POSTs `{ query, session_id }` to `POST /api/query`
2. `backend/app.py` delegates to `RAGSystem.query()`
3. `RAGSystem` (`rag_system.py`) fetches conversation history from `SessionManager`, then calls `AIGenerator`
4. `AIGenerator` (`ai_generator.py`) makes a **first Claude API call** with the query and a `search_course_content` tool available
5. If Claude decides to search, it returns `stop_reason = "tool_use"` — `ToolManager` routes this to `CourseSearchTool`, which runs a semantic search via `VectorStore` (ChromaDB + SentenceTransformer embeddings)
6. Search results are injected as a `tool_result` message and a **second Claude API call** synthesizes the final answer
7. Sources (course/lesson labels tracked by `CourseSearchTool.last_sources`) are returned alongside the answer to the frontend

### Key Components

- **`backend/rag_system.py`** — Central orchestrator. Wires together all components and implements `query()` and `add_course_folder()`.
- **`backend/ai_generator.py`** — Handles all Anthropic API calls. Implements the two-turn tool-use loop in `_handle_tool_execution()`.
- **`backend/vector_store.py`** — ChromaDB wrapper. Maintains two collections: `course_catalog` (course titles/metadata for fuzzy course name resolution) and `course_content` (chunked lesson text for semantic search).
- **`backend/search_tools.py`** — Defines the `search_course_content` tool in Anthropic tool format. `CourseSearchTool` executes searches and tracks sources. `ToolManager` registers and dispatches tools.
- **`backend/document_processor.py`** — Parses `.txt` course files into `Course`/`Lesson`/`CourseChunk` models and splits content into overlapping sentence-based chunks.
- **`backend/session_manager.py`** — In-memory conversation history, keyed by session ID. History is passed to Claude as a formatted string in the system prompt.
- **`backend/models.py`** — Pydantic models: `Course`, `Lesson`, `CourseChunk`.
- **`backend/config.py`** — Reads env vars and exposes a `config` singleton.

### Course Document Format

Documents in `docs/` must follow this structure:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <lesson title>
Lesson Link: <url>
<lesson content...>

Lesson 2: <lesson title>
...
```
