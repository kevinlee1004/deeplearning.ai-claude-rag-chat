# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Retrieval-Augmented Generation (RAG) system for answering questions about course materials. The system uses ChromaDB for vector storage, Anthropic's Claude API for AI generation, and provides a web interface through FastAPI.

## Commands

### Running the Application

**Primary method (from project root):**
```bash
./run.sh
```

**Manual start (from project root):**
```bash
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Installing dependencies:**
```bash
uv sync
```

### Environment Setup

Create a `.env` file in the project root with:
```
ANTHROPIC_API_KEY=your_api_key_here
```

## Architecture

### Core Data Flow

The system follows this request flow:
1. **Frontend** → User submits query via web interface
2. **FastAPI (app.py)** → Receives query, manages sessions
3. **RAGSystem (rag_system.py)** → Orchestrates all components
4. **AIGenerator (ai_generator.py)** → Sends query to Claude with tool definitions
5. **ToolManager/CourseSearchTool (search_tools.py)** → Claude calls search tool when needed
6. **VectorStore (vector_store.py)** → Performs semantic search in ChromaDB
7. **AIGenerator** → Synthesizes final response from search results
8. **FastAPI** → Returns response + sources to frontend

### Key Components

**RAGSystem (rag_system.py)** - Main orchestrator that initializes and coordinates:
- DocumentProcessor: Parses course documents into chunks
- VectorStore: Manages ChromaDB collections and semantic search
- AIGenerator: Handles Claude API interactions
- SessionManager: Tracks conversation history
- ToolManager: Manages AI tool calling

**Document Processing Pipeline:**
- Course documents must follow format: Course Title/Link/Instructor on first 3 lines, then "Lesson N: Title" markers
- DocumentProcessor extracts Course and Lesson metadata, chunks content with overlap
- Chunks are embedded and stored in two ChromaDB collections:
  - `course_catalog`: Course/lesson metadata for fuzzy name matching
  - `course_content`: Actual lesson content chunks

**Vector Search Strategy:**
- Two-collection approach enables fuzzy course name matching before content search
- VectorStore.search() resolves approximate course names via semantic search in catalog, then searches content with filters
- Supports filtering by: course_name (fuzzy matched), lesson_number (exact)

**AI Tool Calling Pattern:**
- AIGenerator provides Claude with search_course_content tool definition
- Claude autonomously decides when to search and with what parameters
- Tool execution happens via ToolManager, results fed back to Claude
- Sources tracked through last_sources mechanism in CourseSearchTool

**Configuration (config.py):**
- CHUNK_SIZE: 800 chars (balance between context and granularity)
- CHUNK_OVERLAP: 100 chars (ensures coherence across chunk boundaries)
- MAX_RESULTS: 5 (top semantic matches returned)
- MAX_HISTORY: 2 (conversation pairs kept for context)
- ANTHROPIC_MODEL: claude-sonnet-4-20250514

### Frontend-Backend Integration

- Frontend served as static files from `frontend/` directory via FastAPI
- API endpoints under `/api/` prefix:
  - POST `/api/query`: Submit question, returns answer + sources + session_id
  - GET `/api/courses`: Get course statistics and titles
- CORS enabled for all origins (development mode)
- DevStaticFiles class adds no-cache headers for development

### Document Loading

On startup, app.py automatically loads all documents from `../docs` folder:
- Checks for existing courses to avoid duplicates
- Processes .pdf, .docx, .txt files
- Each course stored once by title (title is unique ID)

### Models (models.py)

Data structures:
- **Course**: title (unique ID), course_link, instructor, lessons[]
- **Lesson**: lesson_number, title, lesson_link
- **CourseChunk**: content, course_title, lesson_number, chunk_index

## Development Notes

- Uses `uv` as the Python package manager (not pip)
- Python 3.13+ required
- Backend runs from `backend/` directory but paths reference `../docs` and `../frontend`
- ChromaDB persists to `./chroma_db` (relative to backend directory)
- Windows users should use Git Bash to run shell scripts
