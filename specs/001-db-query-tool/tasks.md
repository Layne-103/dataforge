# Tasks: Database Query Tool

**Input**: Design documents from `/specs/001-db-query-tool/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml

**Organization**: Tasks organized into 4 phases as requested - Setup, Core Foundation (P1+P2), Advanced Features (P3+P4), and Polish.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- All file paths are absolute from repository root

## Path Conventions

- **Backend**: `backend/src/`, `backend/tests/`
- **Frontend**: `frontend/src/`, `frontend/tests/`

---

## Phase 1: Project Setup & Infrastructure

**Purpose**: Initialize both backend and frontend projects with all required dependencies and configuration

- [ ] T001 Create backend project structure at backend/ with src/, tests/, pyproject.toml per plan.md
- [ ] T002 Initialize backend with uv: create virtual environment and install dependencies (fastapi, pydantic, sqlglot, openai, sqlalchemy, asyncpg, aiosqlite, openpyxl)
- [ ] T003 [P] Create frontend project structure at frontend/ with src/, tests/, package.json per plan.md
- [ ] T004 [P] Initialize frontend with npm: install dependencies (react, refine, antd, tailwind, monaco-editor, axios, vite)
- [ ] T005 [P] Configure backend linting and formatting: add ruff configuration to pyproject.toml
- [ ] T006 [P] Configure frontend TypeScript: create tsconfig.json with strict: true per constitution
- [ ] T007 [P] Configure Tailwind CSS: create tailwind.config.js and frontend/src/styles/index.css
- [ ] T008 Create backend/src/config.py with AppConfig using pydantic-settings for environment variables (OPENAI_API_KEY, DATAFORGE_SQLITE_DB_PATH)
- [ ] T009 Create backend/src/db/sqlite_manager.py to initialize SQLite schema at ~/.dataforge.db (databases table, query_history table)
- [ ] T010 [P] Create shared/README.md documenting CORS configuration and API versioning standards

**Checkpoint**: Both projects initialized with all dependencies, ready for implementation

---

## Phase 2: Core Foundation - Database Connection & Query Execution (P1 + P2)

**Purpose**: Implement the core MVP - users can connect databases, view metadata, and execute SQL queries

**Goal**: Users can add a database, view its structure, write SQL queries, and see results

**Independent Test**: Add PostgreSQL database → view tables/columns → execute SELECT query → see results in table

### Backend - Data Models & Foundation

- [ ] T011 [P] [US1] Create backend/src/models/__init__.py
- [ ] T012 [P] [US1] Create backend/src/models/database.py with DatabaseConnection, DatabaseMetadata, TableMetadata, ColumnMetadata, DatabaseCreateRequest, DatabaseListResponse (all with alias_generator=to_camel)
- [ ] T013 [P] [US2] Create backend/src/models/query.py with QueryRequest, QueryResult (with alias_generator=to_camel)
- [ ] T014 [P] [US1] Create backend/src/services/__init__.py

### Backend - Database Connection Service (US1)

- [ ] T015 [US1] Create backend/src/services/metadata_cache.py to save/retrieve DatabaseMetadata from SQLite using aiosqlite
- [ ] T016 [US1] Create backend/src/services/db_connection.py with async functions to connect to PostgreSQL via asyncpg, query information_schema for tables/views/columns, return DatabaseMetadata
- [ ] T017 [US1] Update backend/src/services/metadata_cache.py to store connection info (name, url, type) and metadata JSON in SQLite databases table

### Backend - Query Execution Service (US2)

- [ ] T018 [US2] Create backend/src/services/query_parser.py using sqlglot to parse SQL, validate it's SELECT only, add LIMIT 1000 if missing, return transformed SQL or validation error
- [ ] T019 [US2] Create backend/src/services/query_executor.py with async function to execute validated SQL via asyncpg, return QueryResult with columns, rows, rowCount, executionTimeMs

### Backend - API Endpoints (US1 + US2)

- [ ] T020 [P] [US1] Create backend/src/api/__init__.py and backend/src/api/v1/__init__.py
- [ ] T021 [US1] Create backend/src/api/v1/databases.py with GET /api/v1/dbs/ (list all), PUT /api/v1/dbs/{name} (add/update with metadata retrieval), GET /api/v1/dbs/{name} (get details)
- [ ] T022 [US2] Create backend/src/api/v1/queries.py with POST /api/v1/dbs/{name}/query endpoint using query_parser and query_executor services

### Backend - Main Application

- [ ] T023 Create backend/src/main.py with FastAPI app, CORS middleware (allow_origins=["*"]), lifespan context for SQLite init, include v1 routers

### Frontend - Core Components (US1 + US2)

- [ ] T024 [P] [US1] Generate frontend/src/types/api.ts TypeScript types from contracts/openapi.yaml using openapi-typescript or manual creation
- [ ] T025 [P] [US1] Create frontend/src/services/api.ts with axios instance (baseURL from VITE_API_URL environment variable)
- [ ] T026 [P] [US1] Create frontend/src/services/dataProvider.ts implementing Refine data provider using api.ts
- [ ] T027 [US1] Create frontend/src/components/DatabaseList.tsx to display list of databases using Refine useTable hook
- [ ] T028 [US1] Create frontend/src/components/DatabaseMetadata.tsx to show tables/views/columns for a selected database
- [ ] T029 [US1] Create frontend/src/pages/databases/list.tsx with database list page (Refine resource)
- [ ] T030 [US1] Create frontend/src/pages/databases/create.tsx with form to add database connection (name + URL input, calls PUT /dbs/{name})
- [ ] T031 [US2] Create frontend/src/components/SQLEditor.tsx wrapping Monaco Editor with SQL language support, table/column autocomplete from metadata
- [ ] T032 [US2] Create frontend/src/components/QueryResults.tsx to display query results in Ant Design Table with columns from QueryResult
- [ ] T033 [US2] Create frontend/src/pages/databases/show.tsx combining DatabaseMetadata, SQLEditor, QueryResults - execute query on button click

### Frontend - Main Application Setup

- [ ] T034 Create frontend/src/main.tsx with React root, Refine provider, AntdApp wrapper, routerBindings, dataProvider
- [ ] T035 Create frontend/src/App.tsx defining Refine resources (dbs with list/create/show routes), RouterProvider with routes

**Checkpoint**: MVP complete - users can add databases, view metadata, execute SELECT queries, see results

---

## Phase 3: Advanced Features - Natural Language & Export (P3 + P4)

**Purpose**: Add natural language SQL generation and export functionality

**Goal**: Users can generate SQL from natural language prompts and export results to CSV/Excel/JSON

**Independent Test (US3)**: Enter "show all users" → SQL generated → review → execute → see results
**Independent Test (US4)**: Execute query → click Export → select CSV → download file

### Backend - Natural Language to SQL (US3)

- [ ] T036 [P] [US3] Create backend/src/models/query.py additions: NaturalLanguageRequest, NaturalLanguageResponse (with alias_generator=to_camel)
- [ ] T037 [US3] Create backend/src/services/nl_to_sql.py using OpenAI AsyncClient, send metadata as context in system prompt, return generated SQL with explanation using GPT-4
- [ ] T038 [US3] Create backend/src/api/v1/natural.py with POST /api/v1/dbs/{name}/query/natural endpoint calling nl_to_sql service

### Backend - Export Service (US4)

- [ ] T039 [P] [US4] Create backend/src/models/export.py with ExportFormat (enum), ExportRequest, ExportResponse (with alias_generator=to_camel)
- [ ] T040 [US4] Create backend/src/services/export_service.py with to_csv() using csv module, to_excel() using openpyxl, to_json() using json with custom encoder for dates/decimals
- [ ] T041 [US4] Create backend/src/api/v1/exports.py with POST /api/v1/dbs/{name}/export endpoint generating export file, returning download URL with expiry timestamp

### Frontend - Natural Language & Export (US3 + US4)

- [ ] T042 [P] [US3] Create frontend/src/components/NaturalLanguageInput.tsx with text input and "Generate SQL" button calling POST /query/natural
- [ ] T043 [US3] Update frontend/src/pages/databases/show.tsx to add NaturalLanguageInput component, populate SQLEditor with generated SQL for user review
- [ ] T044 [P] [US4] Create frontend/src/components/ExportButton.tsx with Ant Design Dropdown showing CSV/Excel/JSON options, calls POST /export with selected format
- [ ] T045 [US4] Update frontend/src/pages/databases/show.tsx to add ExportButton component below QueryResults, handle download of exported file

**Checkpoint**: All user stories complete - full feature set implemented

---

## Phase 4: Polish & Production Readiness

**Purpose**: Error handling, documentation, validation, and deployment preparation

- [ ] T046 [P] Add comprehensive error handling to all backend endpoints: connection failures, SQL errors, OpenAI errors, export errors - return ErrorResponse with proper codes
- [ ] T047 [P] Update frontend components to display error messages from API using Ant Design notification/message components
- [ ] T048 [P] Create backend/README.md with setup instructions, environment variables, running tests, deployment notes from quickstart.md
- [ ] T049 [P] Create frontend/README.md with setup instructions, environment variables, building for production
- [ ] T050 [P] Add input validation to frontend forms: database URL format validation, non-empty SQL validation, natural language prompt min length
- [ ] T051 Create .env.example files for both backend and frontend documenting required environment variables
- [ ] T052 [P] Run mypy type checking on backend: fix any type errors for strict mode compliance
- [ ] T053 [P] Run TypeScript compiler on frontend: fix any compilation errors for strict mode compliance
- [ ] T054 [P] Run ruff format on backend codebase to ensure consistent formatting
- [ ] T055 [P] Run prettier on frontend codebase to ensure consistent formatting
- [ ] T056 Validate quickstart.md instructions by following them on a clean machine, update any incorrect steps

**Checkpoint**: Production-ready application with proper error handling and documentation

---

## Dependencies & Execution Order

### Phase Dependencies

1. **Phase 1 (Setup)**: No dependencies - start immediately
2. **Phase 2 (Core Foundation)**: Depends on Phase 1 completion
3. **Phase 3 (Advanced Features)**: Depends on Phase 2 completion (needs query execution infrastructure)
4. **Phase 4 (Polish)**: Depends on Phase 2 and 3 completion

### User Story Dependencies

- **US1 (Database Connection)**: Independent - no dependencies on other stories
- **US2 (Query Execution)**: Depends on US1 (needs database metadata for queries)
- **US3 (Natural Language)**: Depends on US1 and US2 (needs metadata and query execution)
- **US4 (Export)**: Depends on US2 (needs query results to export)

### Within Phase 2

Tasks can run in parallel groups:

**Group 1 - Models** (parallel):
- T011-T014: Create all model files

**Group 2 - Services** (sequential within, parallel between US1/US2):
- US1 Services: T015 → T016 → T017 (sequential - cache needs connection service)
- US2 Services: T018, T019 (parallel - parser and executor independent)

**Group 3 - API** (parallel after services):
- T020, T021, T022 (parallel after respective services complete)

**Group 4 - Frontend** (parallel after API):
- T024-T035 (mostly parallel, T034 and T035 depend on components)

### Within Phase 3

Tasks can run in parallel:
- **US3 tasks**: T036-T038, T042-T043 (parallel track)
- **US4 tasks**: T039-T041, T044-T045 (parallel track)
- US3 and US4 can be developed simultaneously by different developers

### Within Phase 4

All polish tasks (T046-T056) can run in parallel as they touch different aspects.

---

## Parallel Execution Examples

### Phase 2 Example: After Models Complete

```bash
# Launch US1 and US2 services in parallel:
Task T015: "Create metadata_cache.py"
Task T016: "Create db_connection.py"  
Task T018: "Create query_parser.py"
Task T019: "Create query_executor.py"
```

### Phase 2 Example: Frontend Components

```bash
# Launch all frontend components together:
Task T027: "Create DatabaseList.tsx"
Task T028: "Create DatabaseMetadata.tsx"
Task T031: "Create SQLEditor.tsx"
Task T032: "Create QueryResults.tsx"
```

### Phase 3 Example: US3 and US4 Together

```bash
# Two developers working in parallel:
Developer A (US3):
  - T036: NaturalLanguage models
  - T037: nl_to_sql service
  - T038: natural.py endpoint
  - T042: NaturalLanguageInput component

Developer B (US4):
  - T039: Export models
  - T040: export_service
  - T041: exports.py endpoint
  - T044: ExportButton component
```

---

## Implementation Strategy

### MVP First (Recommended)

**Phase 1 + Phase 2 = Fully functional MVP**

1. Complete Phase 1: Setup (T001-T010)
2. Complete Phase 2: Core Foundation (T011-T035)
3. **STOP and VALIDATE**:
   - Add test database via UI
   - View tables and columns
   - Execute SELECT query
   - See results in table
4. Deploy MVP if ready

This gives you a working database query tool with core value.

### Incremental Delivery

1. **MVP**: Phase 1 + Phase 2 → Deploy
2. **Enhancement 1**: Add Phase 3 US3 (Natural Language) → Deploy
3. **Enhancement 2**: Add Phase 3 US4 (Export) → Deploy
4. **Production Release**: Add Phase 4 (Polish) → Deploy

Each deployment adds value without breaking previous features.

### Parallel Team Strategy

With 2-3 developers:

1. **Together**: Complete Phase 1 Setup
2. **Split for Phase 2**:
   - Dev A: Backend (T011-T023)
   - Dev B: Frontend (T024-T035)
3. **Split for Phase 3**:
   - Dev A: US3 Natural Language (T036-T038, T042-T043)
   - Dev B: US4 Export (T039-T041, T044-T045)
4. **Together**: Phase 4 Polish (divide T046-T056)

---

## Task Summary

**Total Tasks**: 56
- Phase 1 (Setup): 10 tasks
- Phase 2 (Core P1+P2): 25 tasks
- Phase 3 (Advanced P3+P4): 10 tasks
- Phase 4 (Polish): 11 tasks

**Tasks by User Story**:
- US1 (Database Connection): 14 tasks
- US2 (Query Execution): 10 tasks
- US3 (Natural Language): 6 tasks
- US4 (Export): 6 tasks
- Infrastructure/Polish: 20 tasks

**Parallel Opportunities**: 35 tasks marked [P] can run in parallel

**Estimated Timeline**:
- Phase 1: 1-2 days
- Phase 2: 7-10 days (MVP complete)
- Phase 3: 4-6 days
- Phase 4: 2-3 days
- **Total**: 14-21 days

---

## Notes

- All tasks follow constitution requirements: Python backend with type hints, TypeScript frontend with strict mode, Pydantic models with camelCase output
- Tasks are organized into 4 phases as requested for simplified workflow
- Phase 2 combines US1 and US2 as they form the core MVP and share infrastructure
- Phase 3 combines US3 and US4 as both are enhancement features
- Each task includes specific file paths for clarity
- Tasks marked [P] can be executed in parallel
- Stop after Phase 2 for minimal viable product
- Avoid vague tasks - each specifies exact files and implementation requirements
