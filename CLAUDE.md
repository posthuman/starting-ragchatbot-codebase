# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a RAG (Retrieval-Augmented Generation) chatbot system that enables semantic search over course materials using ChromaDB vector storage and Anthropic's Claude API. The system uses an **agentic tool-calling pattern** where Claude autonomously decides when to search the knowledge base.

## Development Commands

### Setup
```bash
# Install dependencies
uv sync

# Create environment file
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

Access:
- Web interface: http://localhost:8000
- API docs: http://localhost:8000/docs

### Development
```bash
# Run with auto-reload for development
cd backend && uv run uvicorn app:app --reload --port 8000

# Run Python scripts directly
uv run python backend/script_name.py
```

## Architecture Overview

### Core Pattern: Agentic Tool-Calling RAG

The system uses a **two-round Claude API call pattern** for queries that require search:

1. **Round 1**: User query → Claude receives query + tool definitions → Claude decides to use `search_course_content` tool
2. **Round 2**: Tool results → Claude synthesizes results → Final answer to user

This happens automatically when Claude determines a query needs course-specific information.

### Component Interaction Flow

```
User Query (frontend/script.js)
    ↓
FastAPI Endpoint (backend/app.py)
    ↓
RAGSystem Orchestrator (backend/rag_system.py)
    ↓
├─→ SessionManager: Retrieve conversation history
├─→ AIGenerator: Generate response with tool access
│       ↓
│   ┌─→ ToolManager: Execute search_course_content
│   │       ↓
│   │   VectorStore: Semantic search in ChromaDB
│   │       ↓
│   │   Return formatted results
│   │
│   └─→ Claude API Call #2: Synthesize final answer
│
└─→ SessionManager: Store exchange for future context
```

### Key Components

**backend/rag_system.py** - Central orchestrator
- Coordinates all components (document processor, vector store, AI generator, session manager, tools)
- Main entry point: `query(query, session_id)` method
- Handles document ingestion via `add_course_folder()`

**backend/ai_generator.py** - Claude API wrapper
- Implements two-round tool-calling pattern in `_handle_tool_execution()`
- System prompt instructs Claude on search tool usage (lines 8-30)
- Temperature set to 0 for consistency

**backend/vector_store.py** - ChromaDB interface
- **Two collections**: `course_catalog` (metadata) and `course_content` (searchable chunks)
- `search()` method handles semantic course name resolution + filtering
- Uses `all-MiniLM-L6-v2` embeddings via SentenceTransformer

**backend/search_tools.py** - Tool definition and execution
- `CourseSearchTool`: Defines tool schema for Claude
- `ToolManager`: Registers tools, executes them, tracks sources
- Sources stored in `last_sources[]` for UI display

**backend/document_processor.py** - Document parsing and chunking
- Expects format: Course metadata (3 lines) → Lesson markers → Content
- `chunk_text()`: Sentence-based chunking with overlap (800 chars, 100 overlap)
- Adds context prefixes: "Course {title} Lesson {num} content: {chunk}"

**backend/session_manager.py** - Conversation state
- Maintains last 2 Q&A exchanges per session (configurable via `MAX_HISTORY`)
- History injected into system prompt for multi-turn context

**backend/app.py** - FastAPI server
- Serves both API (`/api/query`, `/api/courses`) and static frontend
- Auto-loads documents from `../docs` on startup

### Configuration (backend/config.py)

All tunable parameters are centralized:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2"
- `CHUNK_SIZE`: 800 chars
- `CHUNK_OVERLAP`: 100 chars
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges

### Document Format Requirements

Course documents in `docs/` must follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [lesson title]
Lesson Link: [optional url]
[lesson content...]

Lesson 1: [next lesson]
...
```

The parser uses regex to detect lesson markers (`Lesson \d+:`) and extracts metadata from the first 3 lines.

## Critical Implementation Details

### Tool Calling Pattern
- System prompt (ai_generator.py:8-30) instructs "one search per query maximum"
- Tools are **only available in Round 1** API call to prevent loops
- Round 2 call receives tool results but no tool definitions

### Vector Search Flow
1. If `course_name` provided → `_resolve_course_name()` uses semantic search on `course_catalog` to find best match
2. Build ChromaDB filter from resolved course title and/or lesson number
3. Search `course_content` collection with query embedding
4. Return top `MAX_RESULTS` chunks with metadata

### Session Persistence
- Sessions created on first query if no `session_id` provided
- History formatted as: "User: {query}\nAssistant: {response}"
- Trimmed to last `MAX_HISTORY * 2` messages (default: 4 messages = 2 exchanges)

### ChromaDB Collections
- **course_catalog**: One document per course, ID = course title, stores lessons as JSON string
- **course_content**: One document per chunk, ID = `{course_title}_{chunk_index}`

## Adding New Features

### Adding a New Tool
1. Create tool class in `backend/search_tools.py` inheriting from `Tool`
2. Implement `get_tool_definition()` returning Anthropic tool schema
3. Implement `execute(**kwargs)` with tool logic
4. Register in `rag_system.py`: `self.tool_manager.register_tool(YourTool())`

### Modifying Search Behavior
- Change chunking: Edit `CHUNK_SIZE`/`CHUNK_OVERLAP` in config.py
- Change results count: Edit `MAX_RESULTS` in config.py
- Change embedding model: Edit `EMBEDDING_MODEL` in config.py (requires re-indexing)

### Changing Conversation Context
- Adjust `MAX_HISTORY` in config.py (default: 2 exchanges)
- History formatting in `session_manager.py:42-56`

## Data Flow for New Documents

When adding documents to `docs/`:
1. Restart server (or call `/api/add_course` if endpoint exists)
2. `app.py:startup_event()` calls `rag_system.add_course_folder()`
3. Each `.txt` file → `document_processor.process_course_document()`
4. Course metadata → `vector_store.add_course_metadata()` → `course_catalog` collection
5. Text chunks → `vector_store.add_course_content()` → `course_content` collection with embeddings

Existing courses are skipped based on title matching.

## Environment Requirements

- Python ≥3.13 (specified in pyproject.toml)
- `uv` package manager for dependency management
- Anthropic API key in `.env` file
- Git Bash on Windows (for running shell scripts)
- always yse uv to run server do not use pip directly
- make supre to use uv to manage all dependencies
- use uv to run Python files