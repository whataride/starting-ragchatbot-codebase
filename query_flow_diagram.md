# RAG System Query Processing Flow

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend<br/>(script.js)
    participant API as FastAPI<br/>(app.py)
    participant RAG as RAG System<br/>(rag_system.py)
    participant SM as Session Manager<br/>(session_manager.py)
    participant AI as AI Generator<br/>(ai_generator.py)
    participant TM as Tool Manager<br/>(search_tools.py)
    participant VS as Vector Store<br/>(vector_store.py)
    participant DB as ChromaDB
    participant Claude as Anthropic Claude

    %% User Input
    U->>F: Types query & clicks send
    F->>F: Disable input, show loading
    
    %% API Request
    F->>+API: POST /api/query<br/>{query, session_id}
    API->>API: Validate request
    
    %% Session Management
    API->>SM: Create/get session
    SM-->>API: session_id
    
    %% RAG Processing
    API->>+RAG: query(query, session_id)
    RAG->>SM: Get conversation history
    SM-->>RAG: previous_messages[]
    
    %% AI Generation with Tools
    RAG->>+AI: generate_response(query, history, tools)
    AI->>AI: Build system prompt + context
    AI->>+Claude: API call with tools available
    
    %% Tool Decision & Execution
    Claude-->>AI: Response with tool_use
    AI->>AI: Handle tool execution
    AI->>+TM: execute_tool("search_course_content", params)
    
    %% Vector Search
    TM->>+VS: search(query, course_name, lesson_number)
    VS->>VS: Resolve course name (fuzzy match)
    VS->>VS: Build metadata filters
    VS->>+DB: query(embeddings, filters)
    DB-->>-VS: search_results[]
    VS->>VS: Format results with context
    VS-->>-TM: SearchResults object
    
    %% Tool Response
    TM->>TM: Format & store sources
    TM-->>-AI: formatted_search_results
    AI->>+Claude: Second API call with tool results
    Claude-->>-AI: Final response text
    AI-->>-RAG: Generated answer
    
    %% Source & History Management
    RAG->>TM: get_last_sources()
    TM-->>RAG: sources[]
    RAG->>SM: add_exchange(query, response)
    RAG-->>-API: (answer, sources)
    
    %% API Response
    API->>API: Build QueryResponse
    API-->>-F: {answer, sources, session_id}
    
    %% UI Update
    F->>F: Remove loading animation
    F->>F: Display answer + sources
    F->>U: Show response with expandable sources
```

## Key Components Explained

### Frontend (script.js)
- Handles user input and UI updates
- Makes API calls and manages loading states
- Displays responses with collapsible source citations

### FastAPI (app.py) 
- REST API endpoint `/api/query`
- Request validation and session management
- Orchestrates RAG system calls

### RAG System (rag_system.py)
- Main coordinator that ties all components together
- Manages conversation context and tool integration
- Handles response assembly and source tracking

### AI Generator (ai_generator.py)
- Interface to Anthropic Claude API
- Handles tool calling workflow
- Manages multi-turn conversations with tools

### Tool Manager & Search Tools (search_tools.py)
- Implements course content search capability
- Provides structured tool definitions for Claude
- Formats search results and tracks sources

### Vector Store (vector_store.py)
- ChromaDB integration for semantic search
- Course name resolution and filtering
- Embedding-based content retrieval

### Session Manager (session_manager.py)
- Maintains conversation history
- Provides context for follow-up questions
- Manages session lifecycle

## Data Flow Summary

1. **Input**: User query → Frontend
2. **Transport**: HTTP POST → FastAPI 
3. **Orchestration**: RAG System coordinates all components
4. **Context**: Session Manager provides conversation history
5. **AI Processing**: Claude decides whether to use search tools
6. **Search**: Vector Store performs semantic search in ChromaDB
7. **Synthesis**: Claude generates final response using search results
8. **Assembly**: RAG System combines response with source citations
9. **Output**: Frontend displays answer with expandable sources

The system uses a tool-calling architecture where Claude autonomously decides when to search for information, enabling intelligent responses that combine general knowledge with specific course content.