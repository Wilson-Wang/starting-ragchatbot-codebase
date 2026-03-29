# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Retrieval-Augmented Generation (RAG) system for querying course materials. Uses ChromaDB for vector storage, Anthropic Claude for AI responses, and a vanilla JS frontend.

## Running the Application

```bash
# Install dependencies
uv sync

# Set up environment
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY

# Run the server (always use uv, not pip)
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app is at `http://localhost:8000`. The API docs are at `http://localhost:8000/docs`.

## Key Commands

- `uv sync` - Install/update all dependencies (always use uv, never pip)
- `uv add <package>` - Add a new dependency
- `uv run uvicorn app:app --reload --port 8000` - Run the server from `backend/` directory

## Architecture

### Backend Structure (`backend/`)

- **`app.py`** - FastAPI application. Mounts frontend static files, enables CORS for proxy, registers startup event to load docs from `../docs`
- **`rag_system.py`** - Main orchestrator. Coordinates `DocumentProcessor`, `VectorStore`, `AIGenerator`, `SessionManager`, and `ToolManager`. Exposes `add_course_document()`, `add_course_folder()`, `query()`, `get_course_analytics()`
- **`vector_store.py`** - ChromaDB wrapper. Two collections:
  - `course_catalog` - Course metadata (title, instructor, lessons)
  - `course_content` - Actual course content chunks for semantic search
- **`ai_generator.py`** - Claude API client. Uses `anthropic` SDK with tool support. System prompt instructs Claude to use `search_course_content` tool for course-specific questions
- **`search_tools.py`** - Defines `CourseSearchTool` (implements Tool interface) and `ToolManager`. CourseSearchTool executes searches against the vector store
- **`document_processor.py`** - Parses course documents in expected format and chunks text for embedding
- **`session_manager.py`** - In-memory session storage for conversation history (max history configurable)
- **`models.py`** - Pydantic models: `Course`, `Lesson`, `CourseChunk`
- **`config.py`** - Configuration dataclass. Reads from environment variables and `.env` file

### Frontend (`frontend/`)

- Vanilla JS with no framework. `script.js` handles chat UI and API calls to `/api/query` and `/api/courses`

### Data Flow

1. User query → `app.py` `/api/query` endpoint
2. `RAGSystem.query()` sends to `AIGenerator.generate_response()` with tool definitions
3. If Claude decides to use tools, `ToolManager.execute_tool()` runs `CourseSearchTool.execute()`
4. `CourseSearchTool` queries `VectorStore` which searches ChromaDB
5. Results returned to Claude → final response sent to user

### Course Document Format

Documents in `docs/` should follow this format:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 1: [title]
Lesson Link: [url]
[Content...]

Lesson 2: [title]
[Content...]
```

### ChromaDB Storage

Data persists at `backend/chroma_db/` (gitignored). On startup, `add_course_folder("../docs")` loads documents - skips courses already in the store.

## API Endpoints

- `POST /api/query` - Query course materials. Request: `{query: string, session_id?: string}`. Response: `{answer: string, sources: string[], session_id: string}`
- `GET /api/courses` - Get catalog stats. Response: `{total_courses: int, course_titles: string[]}`
