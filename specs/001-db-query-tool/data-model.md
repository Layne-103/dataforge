# Data Model: Database Query Tool

**Phase**: 1 (Design & Contracts)
**Date**: 2026-01-15
**Purpose**: Define all data entities, validation rules, and relationships

## Overview

This document defines all Pydantic models used in the backend API. All models follow the constitution requirements:
- Use `alias_generator=to_camel` for camelCase JSON output
- Include comprehensive type annotations
- Provide Field descriptions for documentation
- Enable OpenAPI schema generation

## Core Models

### DatabaseConnection

Represents a user-added database connection with metadata cache.

```python
from pydantic import BaseModel, ConfigDict, Field, HttpUrl
from pydantic.alias_generators import to_camel
from datetime import datetime
from typing import Optional

class DatabaseConnection(BaseModel):
    """Database connection configuration and status."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
        json_schema_extra={
            "example": {
                "name": "production_db",
                "connectionUrl": "postgresql://user:pass@localhost:5432/mydb",
                "databaseType": "postgresql",
                "createdAt": "2026-01-15T10:00:00Z",
                "lastConnectedAt": "2026-01-15T10:05:00Z"
            }
        }
    )
    
    id: Optional[int] = Field(None, description="Internal database ID")
    name: str = Field(..., min_length=1, max_length=255, description="Unique name for this database connection")
    connection_url: str = Field(..., description="Database connection URL (e.g., postgresql://user:pass@host:port/dbname)")
    database_type: str = Field(default="postgresql", description="Database type (postgresql, mysql, sqlite)")
    created_at: Optional[datetime] = Field(None, description="When this connection was first added")
    updated_at: Optional[datetime] = Field(None, description="When this connection was last modified")
    last_connected_at: Optional[datetime] = Field(None, description="When we last successfully connected to this database")
    metadata: Optional['DatabaseMetadata'] = Field(None, description="Cached database structure metadata")

class DatabaseCreateRequest(BaseModel):
    """Request to add a new database connection."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    url: str = Field(..., description="Database connection URL", examples=["postgresql://user:pass@localhost:5432/mydb"])

class DatabaseListResponse(BaseModel):
    """Response containing list of all database connections."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    databases: list[DatabaseConnection] = Field(default_factory=list, description="List of all database connections")
    total: int = Field(..., description="Total number of databases")
```

**Validation Rules**:
- `name`: Must be unique, 1-255 characters, alphanumeric with underscores/hyphens
- `connection_url`: Must be valid database URL format
- `database_type`: Enum of supported types (postgresql, mysql, sqlite)

**State Transitions**:
1. Created → metadata is None
2. Connected → metadata is populated from database
3. Cached → metadata retrieved from SQLite
4. Error → last_connected_at not updated

### DatabaseMetadata

Contains structural information about a connected database.

```python
from typing import List, Optional

class ColumnMetadata(BaseModel):
    """Metadata for a single table column."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    name: str = Field(..., description="Column name")
    data_type: str = Field(..., description="SQL data type (e.g., VARCHAR, INTEGER, TIMESTAMP)")
    is_nullable: bool = Field(..., description="Whether column allows NULL values")
    column_default: Optional[str] = Field(None, description="Default value expression if defined")
    character_maximum_length: Optional[int] = Field(None, description="Max length for character types")
    ordinal_position: int = Field(..., description="Column position in table (1-based)")

class TableMetadata(BaseModel):
    """Metadata for a table or view."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    name: str = Field(..., description="Table or view name")
    type: str = Field(..., description="Object type: TABLE, VIEW, MATERIALIZED VIEW")
    columns: List[ColumnMetadata] = Field(default_factory=list, description="Column definitions")
    row_count: Optional[int] = Field(None, description="Approximate row count (for tables)")

class DatabaseMetadata(BaseModel):
    """Complete database structure metadata."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    schema_name: str = Field(default="public", description="Database schema name")
    tables: List[TableMetadata] = Field(default_factory=list, description="All tables in schema")
    views: List[TableMetadata] = Field(default_factory=list, description="All views in schema")
    retrieved_at: datetime = Field(..., description="When this metadata was retrieved")
    table_count: int = Field(..., description="Total number of tables")
    view_count: int = Field(..., description="Total number of views")
```

**Validation Rules**:
- `columns`: Must have at least 1 column per table
- `ordinal_position`: Must be unique within table, starting from 1
- `retrieved_at`: Timestamp of metadata extraction

**Relationships**:
- DatabaseConnection (1) → (1) DatabaseMetadata
- TableMetadata (1) → (N) ColumnMetadata

### QueryRequest & QueryResult

SQL query execution request and response.

```python
from typing import List, Any, Optional

class QueryRequest(BaseModel):
    """Request to execute a SQL query."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    sql: str = Field(..., min_length=1, description="SQL query to execute (SELECT only)", examples=["SELECT * FROM users LIMIT 10"])

class QueryResult(BaseModel):
    """Result of SQL query execution."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    columns: List[str] = Field(..., description="Column names in result set")
    rows: List[List[Any]] = Field(..., description="Result rows as arrays of values")
    row_count: int = Field(..., description="Number of rows returned")
    execution_time_ms: int = Field(..., description="Query execution time in milliseconds")
    transformed_sql: Optional[str] = Field(None, description="Actual SQL executed (with LIMIT added if needed)")
    warnings: List[str] = Field(default_factory=list, description="Non-fatal warnings (e.g., LIMIT added)")
```

**Validation Rules**:
- `sql`: Must parse as valid SQL via sqlglot
- `sql`: Must be SELECT statement only (no INSERT/UPDATE/DELETE/DROP)
- Result set: Limited to 1000 rows (LIMIT enforcement)

**Data Flow**:
1. Frontend sends QueryRequest with user's SQL
2. Backend validates with sqlglot parser
3. Backend adds LIMIT 1000 if missing
4. Backend executes against target database
5. Backend returns QueryResult with camelCase fields

### NaturalLanguageRequest

Natural language to SQL generation request and response.

```python
class NaturalLanguageRequest(BaseModel):
    """Request to generate SQL from natural language."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    prompt: str = Field(..., min_length=5, description="Natural language description of desired query", examples=["Show all users created in the last 30 days"])

class NaturalLanguageResponse(BaseModel):
    """Response with generated SQL and explanation."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    sql: str = Field(..., description="Generated SQL query")
    explanation: str = Field(..., description="Human-readable explanation of what the query does")
    tables_used: List[str] = Field(default_factory=list, description="Tables referenced in the generated query")
    confidence: Optional[float] = Field(None, ge=0.0, le=1.0, description="Confidence score from LLM (if available)")
```

**Validation Rules**:
- `prompt`: Minimum 5 characters to avoid trivial/unclear requests
- Generated `sql`: Must pass same sqlglot validation as manual queries
- `confidence`: Optional 0.0-1.0 range (LLM-provided if available)

**Data Flow**:
1. Frontend sends NaturalLanguageRequest
2. Backend loads database metadata from cache
3. Backend calls OpenAI API with metadata context
4. Backend validates generated SQL with sqlglot
5. Backend returns NaturalLanguageResponse
6. Frontend displays SQL in editor for user review
7. User approves → execute via QueryRequest flow

### ExportRequest & ExportOperation

Export query results to file formats.

```python
from enum import Enum

class ExportFormat(str, Enum):
    """Supported export file formats."""
    CSV = "csv"
    EXCEL = "excel"
    JSON = "json"

class ExportRequest(BaseModel):
    """Request to export query results."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    format: ExportFormat = Field(..., description="Export file format")
    query_result: QueryResult = Field(..., description="Query result to export")
    filename: Optional[str] = Field(None, description="Custom filename (without extension)")

class ExportResponse(BaseModel):
    """Response with export download information."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    download_url: str = Field(..., description="Temporary URL to download the exported file")
    filename: str = Field(..., description="Generated filename with extension")
    format: ExportFormat = Field(..., description="File format")
    size_bytes: int = Field(..., description="File size in bytes")
    expires_at: datetime = Field(..., description="When the download URL expires")
```

**Validation Rules**:
- `format`: Must be one of CSV, EXCEL, JSON
- `filename`: If provided, must be valid filename (alphanumeric, underscores, hyphens)
- Generated filenames: `dataforge_export_{timestamp}.{ext}`

**Export Behavior**:
- CSV: RFC 4180 compliant, UTF-8 encoding, proper escaping of quotes/commas
- Excel: .xlsx format, bold headers, auto-width columns, first row frozen
- JSON: Array of objects with column names as keys, proper date/decimal serialization

## Error Models

### ErrorResponse

Standardized error response for all API errors.

```python
from typing import Optional, Dict, Any

class ErrorDetail(BaseModel):
    """Detailed error information."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    field: Optional[str] = Field(None, description="Field name that caused the error (if applicable)")
    message: str = Field(..., description="Human-readable error message")
    code: str = Field(..., description="Machine-readable error code")

class ErrorResponse(BaseModel):
    """Standard error response."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    error: str = Field(..., description="Error type or category")
    message: str = Field(..., description="Primary error message")
    details: Optional[List[ErrorDetail]] = Field(None, description="Additional error details")
    request_id: Optional[str] = Field(None, description="Request ID for debugging")
```

**Common Error Codes**:
- `DB_CONNECTION_FAILED`: Cannot connect to database
- `INVALID_SQL_SYNTAX`: SQL parsing failed
- `FORBIDDEN_SQL_STATEMENT`: Non-SELECT statement detected
- `QUERY_EXECUTION_ERROR`: Database returned error during execution
- `LLM_GENERATION_FAILED`: OpenAI API error or invalid response
- `EXPORT_GENERATION_FAILED`: Error creating export file
- `DATABASE_NOT_FOUND`: Referenced database name doesn't exist

## Configuration Models

### AppConfig

Application configuration from environment variables.

```python
from pydantic_settings import BaseSettings

class AppConfig(BaseSettings):
    """Application configuration from environment."""
    
    model_config = ConfigDict(
        env_prefix="DATAFORGE_",
        case_sensitive=False
    )
    
    openai_api_key: str = Field(..., description="OpenAI API key for natural language SQL generation")
    sqlite_db_path: str = Field(default="~/.dataforge.db", description="Path to SQLite metadata cache")
    default_query_limit: int = Field(default=1000, description="Default LIMIT for unbounded queries")
    query_timeout_seconds: int = Field(default=30, description="Maximum query execution time")
    export_url_expiry_minutes: int = Field(default=15, description="Export download URL expiry time")
    cors_origins: List[str] = Field(default=["*"], description="Allowed CORS origins")
    log_level: str = Field(default="INFO", description="Logging level")
```

**Environment Variables**:
- `OPENAI_API_KEY`: Required, no default
- `DATAFORGE_SQLITE_DB_PATH`: Optional, defaults to ~/.dataforge.db
- `DATAFORGE_DEFAULT_QUERY_LIMIT`: Optional, defaults to 1000
- `DATAFORGE_QUERY_TIMEOUT_SECONDS`: Optional, defaults to 30

## Model Relationships Diagram

```
DatabaseConnection (1:1) DatabaseMetadata
       |
       | (1:N)
       |
QueryHistory ──────> QueryResult
       |
       |
NaturalLanguageRequest ──> NaturalLanguageResponse ──> QueryRequest
       |
       |
QueryResult ──────> ExportRequest ──> ExportResponse
```

## Serialization Examples

### Request: Add Database

```json
PUT /api/v1/dbs/my_postgres

{
  "url": "postgresql://user:password@localhost:5432/mydb"
}
```

### Response: Database with Metadata

```json
{
  "id": 1,
  "name": "my_postgres",
  "connectionUrl": "postgresql://user:***@localhost:5432/mydb",
  "databaseType": "postgresql",
  "createdAt": "2026-01-15T10:00:00Z",
  "updatedAt": "2026-01-15T10:05:00Z",
  "lastConnectedAt": "2026-01-15T10:05:00Z",
  "metadata": {
    "schemaName": "public",
    "tables": [
      {
        "name": "users",
        "type": "TABLE",
        "columns": [
          {
            "name": "id",
            "dataType": "INTEGER",
            "isNullable": false,
            "columnDefault": "nextval('users_id_seq')",
            "ordinalPosition": 1
          },
          {
            "name": "email",
            "dataType": "VARCHAR",
            "isNullable": false,
            "characterMaximumLength": 255,
            "ordinalPosition": 2
          }
        ],
        "rowCount": 1500
      }
    ],
    "views": [],
    "retrievedAt": "2026-01-15T10:05:00Z",
    "tableCount": 1,
    "viewCount": 0
  }
}
```

Note: All field names are in camelCase as required by constitution. Backend uses snake_case internally, Pydantic converts on serialization.

## Validation Summary

All models enforce:
1. **Type Safety**: Full type annotations on all fields
2. **camelCase Output**: Via `alias_generator=to_camel`
3. **Runtime Validation**: Pydantic validates on request/response
4. **OpenAPI Generation**: Automatic schema for frontend TypeScript types
5. **Documentation**: Field descriptions for API documentation

Ready for contract generation (OpenAPI specification).
