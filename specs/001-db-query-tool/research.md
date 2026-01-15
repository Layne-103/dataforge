# Research: Database Query Tool

**Phase**: 0 (Outline & Research)
**Date**: 2026-01-15
**Purpose**: Resolve technical unknowns and establish best practices for implementation

## Research Topics

### 1. PostgreSQL Metadata Retrieval

**Decision**: Use PostgreSQL information_schema and pg_catalog system views

**Rationale**:
- `information_schema.tables` and `information_schema.columns` provide standardized metadata access
- `pg_catalog.pg_class` and `pg_catalog.pg_attribute` offer more detailed PostgreSQL-specific information
- Supports views, materialized views, tables with full column metadata (types, constraints, defaults)
- Standard SQL interface compatible with SQLAlchemy

**Implementation Approach**:
```sql
-- Get all tables and views
SELECT table_name, table_type 
FROM information_schema.tables 
WHERE table_schema = 'public';

-- Get column details for a table
SELECT column_name, data_type, is_nullable, column_default, character_maximum_length
FROM information_schema.columns
WHERE table_schema = 'public' AND table_name = ?
ORDER BY ordinal_position;
```

**Alternatives Considered**:
- Direct psycopg3 queries: More control but less portable, harder to test
- ORM introspection (SQLAlchemy): Too heavyweight for metadata-only operations
- pg_dump parsing: Complex, requires external process, fragile

### 2. SQL Parsing and Validation with sqlglot

**Decision**: Use sqlglot for SQL parsing, validation, and manipulation

**Rationale**:
- Pure Python library (no external dependencies like libpq)
- Supports multiple SQL dialects (PostgreSQL, MySQL, SQLite, etc.)
- Can parse, validate, and modify SQL AST (add LIMIT clauses)
- Detects statement types (SELECT vs INSERT/UPDATE/DELETE) reliably
- Active maintenance, good error messages

**Implementation Approach**:
```python
import sqlglot
from sqlglot import parse_one, exp

def validate_and_transform_query(sql: str) -> tuple[str, bool]:
    """
    Validates SQL is SELECT-only and adds LIMIT if missing.
    Returns (transformed_sql, is_valid)
    """
    try:
        parsed = parse_one(sql, dialect="postgres")
        
        # Check if it's a SELECT statement
        if not isinstance(parsed, exp.Select):
            return sql, False
            
        # Check if LIMIT exists
        if not parsed.args.get("limit"):
            # Add LIMIT 1000
            parsed = parsed.limit(1000)
            
        return parsed.sql(dialect="postgres"), True
    except Exception as e:
        return sql, False
```

**Alternatives Considered**:
- pyparsing: Too low-level, requires writing SQL grammar
- sqlparse: Only tokenizes, doesn't build AST or validate semantics
- Regular expressions: Unreliable, easily bypassed with subqueries or comments

### 3. Natural Language to SQL with OpenAI

**Decision**: Use OpenAI GPT-4 with structured metadata context

**Rationale**:
- GPT-4 has strong SQL generation capabilities
- Can include schema information in system prompt for context
- Supports JSON mode for structured output (SQL + explanation)
- OpenAI SDK handles retries, rate limiting, error handling

**Implementation Approach**:
```python
from openai import AsyncOpenAI
import json

async def generate_sql_from_nl(
    prompt: str, 
    metadata: dict,
    client: AsyncOpenAI
) -> dict:
    """
    Generates SQL from natural language using database metadata as context.
    """
    system_message = f"""You are a SQL expert. Generate PostgreSQL queries based on user requests.
    
Available tables and columns:
{json.dumps(metadata, indent=2)}

Rules:
1. Only generate SELECT statements
2. Use proper table and column names from the schema
3. Include appropriate WHERE, JOIN, and ORDER BY clauses
4. Return response as JSON with 'sql' and 'explanation' fields
"""
    
    response = await client.chat.completions.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": system_message},
            {"role": "user", "content": prompt}
        ],
        response_format={"type": "json_object"},
        temperature=0.3  # Lower temperature for more deterministic SQL
    )
    
    return json.loads(response.choices[0].message.content)
```

**Alternatives Considered**:
- Text-to-SQL models (e.g., PICARD, RAT-SQL): Require training/fine-tuning, less flexible
- Anthropic Claude: Good alternative but OpenAI SDK specified in constitution
- Local LLMs (Llama, Mistral): Inference overhead, quality concerns for SQL generation

### 4. FastAPI Best Practices for Database Tool

**Decision**: Use async FastAPI with dependency injection for database connections

**Rationale**:
- Async endpoints prevent blocking on database I/O
- Dependency injection manages database connection lifecycle
- Pydantic v2 for automatic request/response validation
- CORS middleware configured per requirements (allow all origins)

**Implementation Approach**:
```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Initialize SQLite metadata cache
    await init_metadata_db()
    yield
    # Shutdown: Close connections
    await close_metadata_db()

app = FastAPI(
    title="DataForge API",
    version="1.0.0",
    lifespan=lifespan
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Dependency for database operations
async def get_metadata_service() -> MetadataCache:
    return MetadataCache()

@app.put("/api/v1/dbs/{name}")
async def add_database(
    name: str,
    request: DatabaseCreateRequest,
    service: MetadataCache = Depends(get_metadata_service)
):
    # Implementation
    pass
```

**Key Patterns**:
- Use `async def` for all endpoints
- Dependency injection for services (metadata cache, query executor)
- Exception handlers for consistent error responses
- Response models for automatic camelCase serialization

### 5. Export Format Generation

**Decision**: Use specialized libraries per format

**Rationale**:
- CSV: Python stdlib `csv` module with proper escaping
- Excel: `openpyxl` for XLSX generation with formatting
- JSON: Python stdlib `json` with custom encoder for dates/decimals

**Implementation Approach**:
```python
import csv
import io
import json
from datetime import datetime, date
from decimal import Decimal
from openpyxl import Workbook
from openpyxl.styles import Font

class ExportService:
    @staticmethod
    def to_csv(results: QueryResult) -> bytes:
        """Generate CSV with proper escaping."""
        output = io.StringIO()
        writer = csv.writer(output, quoting=csv.QUOTE_MINIMAL)
        
        # Write headers
        writer.writerow(results.columns)
        
        # Write rows
        writer.writerows(results.rows)
        
        return output.getvalue().encode('utf-8')
    
    @staticmethod
    def to_excel(results: QueryResult) -> bytes:
        """Generate Excel with formatted headers."""
        wb = Workbook()
        ws = wb.active
        
        # Write headers with bold font
        for col_idx, col_name in enumerate(results.columns, 1):
            cell = ws.cell(row=1, column=col_idx, value=col_name)
            cell.font = Font(bold=True)
        
        # Write data rows
        for row_idx, row in enumerate(results.rows, 2):
            for col_idx, value in enumerate(row, 1):
                ws.cell(row=row_idx, column=col_idx, value=value)
        
        output = io.BytesIO()
        wb.save(output)
        return output.getvalue()
    
    @staticmethod
    def to_json(results: QueryResult) -> bytes:
        """Generate JSON with proper date/decimal handling."""
        class CustomEncoder(json.JSONEncoder):
            def default(self, obj):
                if isinstance(obj, (datetime, date)):
                    return obj.isoformat()
                if isinstance(obj, Decimal):
                    return float(obj)
                return super().default(obj)
        
        data = {
            "columns": results.columns,
            "rows": results.rows,
            "rowCount": len(results.rows)
        }
        return json.dumps(data, cls=CustomEncoder).encode('utf-8')
```

**Alternatives Considered**:
- pandas: Too heavyweight for simple export operations
- CSV only: User requirement specifies Excel and JSON support
- Streaming exports: Over-engineered for 1000-row limit, can add later if needed

### 6. Frontend State Management with Refine

**Decision**: Use Refine's built-in data provider and state management

**Rationale**:
- Refine provides data provider abstraction over REST APIs
- Built-in caching, loading states, error handling
- Integrates with React Query for efficient data fetching
- No additional state management library needed (Redux, Zustand)

**Implementation Approach**:
```typescript
import { Refine } from "@refinedev/core";
import routerBindings from "@refinedev/react-router-v6";
import dataProvider from "@refinedev/simple-rest";
import { AntdApp } from "@refinedev/antd";

const API_URL = import.meta.env.VITE_API_URL || "http://localhost:8000/api/v1";

function App() {
  return (
    <AntdApp>
      <Refine
        dataProvider={dataProvider(API_URL)}
        routerProvider={routerBindings}
        resources={[
          {
            name: "dbs",
            list: "/databases",
            create: "/databases/create",
            show: "/databases/show/:id",
          },
        ]}
      >
        {/* Routes */}
      </Refine>
    </AntdApp>
  );
}
```

**Key Patterns**:
- Refine resources map to backend API endpoints
- useTable, useForm hooks for data operations
- Automatic loading states, error handling, optimistic updates

### 7. Monaco Editor Integration for SQL

**Decision**: Use @monaco-editor/react with SQL language support

**Rationale**:
- Monaco is VS Code's editor (feature-rich, well-maintained)
- Built-in SQL syntax highlighting and basic autocomplete
- Lightweight wrapper for React integration
- Can extend with custom completions for table/column names

**Implementation Approach**:
```typescript
import Editor from "@monaco-editor/react";

interface SQLEditorProps {
  value: string;
  onChange: (value: string) => void;
  metadata?: DatabaseMetadata;
}

export function SQLEditor({ value, onChange, metadata }: SQLEditorProps) {
  const handleMount = (editor: any, monaco: any) => {
    // Configure SQL language
    monaco.languages.setLanguageConfiguration("sql", {
      wordPattern: /(-?\d*\.\d\w*)|([^\`\~\!\@\#\%\^\&\*\(\)\-\=\+\[\{\]\}\\\|\;\:\'\"\,\.\<\>\/\?\s]+)/g,
    });
    
    // Add custom completions for tables/columns
    if (metadata) {
      monaco.languages.registerCompletionItemProvider("sql", {
        provideCompletionItems: (model: any, position: any) => {
          const suggestions = metadata.tables.map((table) => ({
            label: table.name,
            kind: monaco.languages.CompletionItemKind.Class,
            insertText: table.name,
          }));
          return { suggestions };
        },
      });
    }
  };

  return (
    <Editor
      height="300px"
      language="sql"
      theme="vs-dark"
      value={value}
      onChange={onChange}
      onMount={handleMount}
      options={{
        minimap: { enabled: false },
        fontSize: 14,
        lineNumbers: "on",
        automaticLayout: true,
      }}
    />
  );
}
```

**Alternatives Considered**:
- CodeMirror 6: Good alternative but Monaco more feature-complete
- Plain textarea: Poor UX, no syntax highlighting
- Custom SQL editor: Significant development effort, worse UX

### 8. SQLite Metadata Cache Schema

**Decision**: Simple relational schema with JSON columns for flexibility

**Rationale**:
- SQLite embedded in Python stdlib (no external database)
- Stored at `~/.dataforge.db` per specification
- JSON columns for metadata flexibility (different database types have different metadata)
- Fast lookups by database name

**Schema Design**:
```sql
CREATE TABLE IF NOT EXISTS databases (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL,
    connection_url TEXT NOT NULL,
    database_type TEXT NOT NULL DEFAULT 'postgresql',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_connected_at TIMESTAMP,
    metadata_json TEXT  -- JSON: {tables: [...], views: [...]}
);

CREATE INDEX idx_databases_name ON databases(name);

CREATE TABLE IF NOT EXISTS query_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    database_id INTEGER NOT NULL,
    sql_text TEXT NOT NULL,
    natural_language_prompt TEXT,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    row_count INTEGER,
    execution_time_ms INTEGER,
    FOREIGN KEY (database_id) REFERENCES databases(id)
);

CREATE INDEX idx_query_history_database ON query_history(database_id);
CREATE INDEX idx_query_history_executed ON query_history(executed_at DESC);
```

**Alternatives Considered**:
- Normalized metadata tables: Over-engineered for cache use case
- File-based storage (JSON files): Harder to query, no ACID guarantees
- In-memory only: Loses metadata on restart, defeats caching purpose

## Implementation Priority

Based on user story priorities (P1 → P2 → P3 → P4):

1. **Phase 1 (P1 - Database Connection)**:
   - SQLite metadata cache setup
   - PostgreSQL connection and metadata retrieval
   - FastAPI endpoints: GET /dbs/, PUT /dbs/{name}, GET /dbs/{name}

2. **Phase 2 (P2 - Manual SQL Queries)**:
   - sqlglot parser integration
   - Query executor service
   - FastAPI endpoint: POST /dbs/{name}/query
   - Monaco Editor integration in frontend
   - Query results display component

3. **Phase 3 (P3 - Natural Language SQL)**:
   - OpenAI integration service
   - FastAPI endpoint: POST /dbs/{name}/query/natural
   - Natural language input component

4. **Phase 4 (P4 - Export Results)**:
   - Export service (CSV/Excel/JSON)
   - FastAPI endpoint: POST /dbs/{name}/export
   - Export button component with format selection

## Risk Mitigation

### Risk 1: Database Connection Failures
**Mitigation**: 
- Comprehensive error handling with specific error types
- Connection timeout configuration (5 seconds)
- Clear error messages to users indicating connection issues
- Test with invalid URLs, unreachable hosts, auth failures

### Risk 2: SQL Injection via sqlglot Bypass
**Mitigation**:
- sqlglot validation before execution (whitelist SELECT only)
- Additional runtime check that parsed statement is exp.Select
- No string concatenation for SQL building
- Use parameterized queries for metadata retrieval

### Risk 3: OpenAI API Quota/Rate Limits
**Mitigation**:
- Catch rate limit errors and return user-friendly message
- Consider caching common natural language patterns
- Add retry logic with exponential backoff
- Document OPENAI_API_KEY requirement clearly

### Risk 4: Large Metadata (1000+ Tables)
**Mitigation**:
- Pagination for table list display in frontend
- Lazy-load column metadata (on table selection)
- Cache metadata in SQLite to avoid repeated retrieval
- Add loading indicators for metadata operations

### Risk 5: Excel Export Library Size
**Mitigation**:
- openpyxl is pure Python, acceptable size (~2MB)
- Only imported when export endpoint called (not on startup)
- Consider lazy import to reduce initial memory footprint
- Document as optional dependency if needed

## Conclusion

All technical unknowns resolved. No NEEDS CLARIFICATION markers remaining. Ready to proceed to Phase 1 (Design & Contracts).
