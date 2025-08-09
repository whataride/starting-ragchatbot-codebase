# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Environment Setup
```bash
# Install dependencies
uv sync

# Create .env file with required variables
echo "ANTHROPIC_API_KEY=your_key_here" > .env
```

### Access Points
- Web Interface: http://localhost:8000  
- API Documentation: http://localhost:8000/docs

## Architecture Overview

This is a **Retrieval-Augmented Generation (RAG) system** for course materials with a tool-calling architecture where Claude autonomously decides when to search for information.

### Core Architecture Pattern

The system follows a **layered orchestration pattern**:

1. **RAG System (`rag_system.py`)** - Main orchestrator that coordinates all components
2. **AI Generator (`ai_generator.py`)** - Manages Claude API interactions with tool-calling workflow
3. **Tool Manager (`search_tools.py`)** - Implements search tools that Claude can call
4. **Vector Store (`vector_store.py`)** - ChromaDB integration with semantic search and fuzzy course matching

### Key Architectural Concepts

**Tool-Calling Workflow**: Claude receives tool definitions and autonomously decides when to search. When Claude calls `search_course_content`, the system:
- Executes vector search in ChromaDB
- Returns formatted results to Claude  
- Claude synthesizes final response
- Sources are tracked separately for UI display

**Document Processing Pipeline**: Documents follow a specific structure (`Course Title:`, `Lesson N:`) and are processed into:
- Course metadata (stored in `course_catalog` collection)
- Text chunks with context (stored in `course_content` collection)
- Each chunk gets contextual prefixes like "Course X Lesson Y content:"

**Session Management**: Conversations maintain context through `SessionManager` with configurable history limits (default: 2 messages). Session IDs are created on first query and persist conversation state.

**Fuzzy Course Resolution**: The `vector_store._resolve_course_name()` method enables partial course name matching (e.g., "MCP" matches "Building Towards Computer Use with MCP").

### Data Flow Architecture

```
User Query → FastAPI → RAG System → AI Generator → Claude API
                ↓                      ↓
Session Manager ← ← ← ← Tool Manager ← ← (tool decision)
                ↓                      ↓
Vector Store → ChromaDB → Search Results → Formatted Response
                ↓                      ↓
Sources Tracked ← ← ← ← ← Response + Sources → Frontend
```

### Configuration System

Configuration is centralized in `config.py` with these key settings:
- `CHUNK_SIZE: 800` - Text chunk size for vector storage
- `CHUNK_OVERLAP: 100` - Overlap between chunks for context preservation
- `MAX_RESULTS: 5` - Search results limit
- `MAX_HISTORY: 2` - Conversation context limit

### File Structure Patterns

**Backend Structure**:
- `app.py` - FastAPI server and endpoints
- `rag_system.py` - Main coordinator 
- `ai_generator.py` - Claude API interface
- `search_tools.py` - Tool definitions and execution
- `vector_store.py` - ChromaDB operations
- `document_processor.py` - Document parsing and chunking
- `session_manager.py` - Conversation state
- `models.py` - Pydantic data models
- `config.py` - Configuration management

**Document Format**: Course documents in `docs/` follow this structure:
```
Course Title: [title]
Course Link: [url] 
Course Instructor: [instructor]

Lesson 0: [lesson title]
Lesson Link: [lesson url]
[lesson content]
```

## Working with This Codebase

### Adding New Search Tools
1. Inherit from `Tool` class in `search_tools.py`
2. Implement `get_tool_definition()` and `execute()` methods
3. Register with `ToolManager` in `rag_system.py`

### Modifying Document Processing
- Text chunking logic is in `document_processor.py:chunk_text()`
- Course structure parsing is in `document_processor.py:process_course_document()`
- Chunk size/overlap configured in `config.py`

### Vector Store Operations
- Two ChromaDB collections: `course_catalog` (metadata) and `course_content` (chunks)
- Course name resolution uses semantic search in catalog
- Content search supports course and lesson filtering

### Session Context Management
- Sessions auto-created on first query
- History trimmed to `MAX_HISTORY * 2` messages
- Conversation context passed to Claude as system message
- always use uv to run the server do not use pip directly
- use uv to manage all dependencies