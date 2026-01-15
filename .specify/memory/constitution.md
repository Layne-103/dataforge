<!--
SYNC IMPACT REPORT
==================
Version Change: 1.0.0 (Initial Constitution)
Ratification Date: 2026-01-15
Last Amended: 2026-01-15

Constitution Principles:
- I. Language & Style Standards (NEW)
- II. Type Safety (NEW)
- III. Data Modeling Standards (NEW)
- IV. API Contract Standards (NEW)
- V. Accessibility & Security (NEW)

Templates Status:
✅ plan-template.md - Reviewed (Constitution Check section will reference these principles)
✅ spec-template.md - Reviewed (Requirements section aligns with principles)
✅ tasks-template.md - Reviewed (Task structure supports principle-driven development)

Follow-up Actions:
- None - All principles clearly defined
- No deferred placeholders
-->

# DataForge Constitution

## Core Principles

### I. Language & Style Standards

**Backend**: All backend code MUST be written in Python using an ergonomic, Pythonic style that prioritizes readability and maintainability.

**Frontend**: All frontend code MUST be written in TypeScript with strict mode enabled.

**Rationale**: Python's expressiveness accelerates backend development while maintaining code clarity. TypeScript prevents entire classes of runtime errors through compile-time type checking, essential for complex UI logic.

### II. Type Safety (NON-NEGOTIABLE)

**Strict Type Annotations Required**: Both frontend and backend MUST implement comprehensive type annotations for all functions, variables, and data structures.

- **Backend (Python)**: Use type hints (`def function(param: str) -> int:`) for all function signatures, class attributes, and variables where type inference is ambiguous
- **Frontend (TypeScript)**: Enable `strict: true` in `tsconfig.json`; avoid `any` type except where interfacing with untyped external libraries (must be documented)

**Rationale**: Type safety catches bugs at development time rather than runtime, enables better IDE support, serves as living documentation, and makes refactoring safer across both codebases.

### III. Data Modeling Standards

**Pydantic for Schema Definitions**: All data types, schemas, and validation logic MUST be defined using Pydantic models in the backend.

- Models serve as single source of truth for data structure
- Automatic validation at runtime
- Self-documenting through field types and descriptions
- Enables automatic OpenAPI/JSON Schema generation

**Example**:
```python
from pydantic import BaseModel, Field

class DatabaseConnection(BaseModel):
    id: int
    name: str
    connection_url: str = Field(..., description="Database connection string")
    created_at: datetime
```

**Rationale**: Pydantic provides runtime validation, serialization, and clear contracts between backend services and frontend consumers, reducing integration bugs.

### IV. API Contract Standards

**camelCase for All Backend-Generated Data**: All JSON responses, field names, and API payloads from the backend MUST use camelCase naming convention (e.g., `userId`, `createdAt`, `connectionUrl`).

- Python backend code uses snake_case internally (Pythonic convention)
- Pydantic models configure `alias_generator` to convert snake_case → camelCase for serialization
- Frontend TypeScript receives consistent camelCase data matching JavaScript conventions

**Example Configuration**:
```python
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

class DataModel(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    user_id: int  # Internal: snake_case
    # JSON output: {"userId": 123}
```

**Rationale**: Aligning API output with frontend JavaScript/TypeScript conventions (camelCase) reduces friction, eliminates conversion errors, and makes frontend code more idiomatic. Backend maintains Pythonic style internally while providing frontend-friendly contracts.

### V. Accessibility & Security

**Open Access Policy**: The application MUST NOT require authentication. All features MUST be accessible to any user without login, registration, or access controls.

**Security Considerations**:
- Validate and sanitize all user inputs (SQL queries, database URLs)
- Implement rate limiting to prevent abuse
- Use read-only database connections where possible
- Enforce SQL query restrictions (SELECT-only, limit clauses)
- Do not store sensitive credentials in application database

**Rationale**: For this tool's use case (database exploration and querying), removing authentication barriers maximizes accessibility and reduces deployment complexity. Security is maintained through input validation and query restrictions rather than user authentication.

## Technical Stack Standards

**Backend Technologies**:
- Python (latest stable 3.x)
- FastAPI for web framework
- Pydantic for data validation
- sqlglot for SQL parsing and validation
- OpenAI SDK for LLM integration
- SQLite for metadata storage
- uv for package management

**Frontend Technologies**:
- React for UI framework
- Refine 5 for admin/data management patterns
- Tailwind CSS for styling
- Ant Design for component library
- Monaco Editor for SQL editing

**Rationale**: These choices provide modern, well-maintained tools with strong type support and excellent developer experience while maintaining the constitutional principles above.

## Development Workflow

### Code Quality Gates

1. **Type Checking**: All code MUST pass type checking
   - Backend: `mypy --strict`
   - Frontend: `tsc --noEmit`

2. **Linting**: All code MUST pass linter checks
   - Backend: `ruff check`
   - Frontend: `eslint`

3. **Formatting**: All code MUST be formatted consistently
   - Backend: `ruff format`
   - Frontend: `prettier`

### Testing Requirements

- **SQL Validation**: All SQL queries MUST be parsed by sqlglot before execution
- **Query Restrictions**: Parser MUST reject non-SELECT statements
- **Limit Enforcement**: Queries without LIMIT clause MUST have `LIMIT 1000` added automatically
- **Integration Tests**: Database connection and metadata extraction functionality MUST have integration tests

### API Development Workflow

1. Define Pydantic models for request/response (with camelCase alias configuration)
2. Generate TypeScript types from Pydantic models (via OpenAPI schema)
3. Implement FastAPI endpoint with type-annotated handlers
4. Frontend consumes types and receives camelCase data automatically

## Governance

**Constitution Authority**: This constitution supersedes all other development practices and decisions. Any deviation MUST be explicitly documented with justification and risk assessment.

**Amendment Process**:
1. Propose amendment with clear rationale
2. Document impact on existing code and templates
3. Update constitution version (semantic versioning: MAJOR.MINOR.PATCH)
4. Update all dependent templates and documentation
5. Create migration plan for existing code if needed

**Compliance Review**:
- All pull requests MUST verify compliance with constitutional principles
- All feature specifications MUST reference applicable principles
- All implementation plans MUST include "Constitution Check" section

**Versioning Policy**:
- **MAJOR**: Backward-incompatible principle changes (e.g., language switch, authentication requirement added)
- **MINOR**: New principles added or existing principles materially expanded
- **PATCH**: Clarifications, wording improvements, non-semantic refinements

**Version**: 1.0.0 | **Ratified**: 2026-01-15 | **Last Amended**: 2026-01-15
