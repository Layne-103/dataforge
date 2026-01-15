# Quickstart: Database Query Tool

**Purpose**: Step-by-step guide to set up, run, and validate the database query tool

**Audience**: Developers setting up the project for the first time

## Prerequisites

- Python 3.13+ installed
- Node.js 18+ and npm installed
- PostgreSQL database accessible for testing (can use Docker)
- OpenAI API key

## Quick Start (5 minutes)

### 1. Clone and Setup

```bash
# Clone repository
git clone <repository-url>
cd dataforge

# Checkout feature branch
git checkout 001-db-query-tool
```

### 2. Backend Setup

```bash
cd backend

# Install uv package manager (if not installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create virtual environment and install dependencies
uv venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
uv pip install -e .

# Set environment variables
export OPENAI_API_KEY="sk-your-key-here"
export DATAFORGE_SQLITE_DB_PATH="~/.dataforge.db"

# Run database migrations (creates SQLite schema)
python -m src.db.sqlite_manager init

# Start backend server
uvicorn src.main:app --reload --port 8000
```

Backend should now be running at `http://localhost:8000`

### 3. Frontend Setup

```bash
# Open new terminal
cd frontend

# Install dependencies
npm install

# Create .env file
cat > .env << EOF
VITE_API_URL=http://localhost:8000/api/v1
EOF

# Start frontend development server
npm run dev
```

Frontend should now be running at `http://localhost:5173`

### 4. Add Test Database

Open browser to `http://localhost:5173` and:

1. Click "Add Database"
2. Enter name: `test_db`
3. Enter URL: `postgresql://user:password@localhost:5432/testdb`
4. Click "Save"
5. Verify tables/views appear in metadata panel

### 5. Run Test Query

1. Select `test_db` from database list
2. In SQL editor, type: `SELECT * FROM users LIMIT 5`
3. Click "Execute"
4. Verify results appear in table format
5. Try export: Click "Export" → Select "CSV" → Download should start

## Detailed Setup

### Backend Environment Variables

Create `backend/.env`:

```bash
# Required
OPENAI_API_KEY=sk-your-openai-api-key

# Optional (defaults shown)
DATAFORGE_SQLITE_DB_PATH=~/.dataforge.db
DATAFORGE_DEFAULT_QUERY_LIMIT=1000
DATAFORGE_QUERY_TIMEOUT_SECONDS=30
DATAFORGE_EXPORT_URL_EXPIRY_MINUTES=15
DATAFORGE_CORS_ORIGINS=["*"]
DATAFORGE_LOG_LEVEL=INFO
```

### Backend Dependencies

The `pyproject.toml` includes:

```toml
[project]
name = "dataforge"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "fastapi>=0.104.0",
    "uvicorn[standard]>=0.24.0",
    "pydantic>=2.5.0",
    "pydantic-settings>=2.1.0",
    "sqlalchemy>=2.0.0",
    "asyncpg>=0.29.0",  # PostgreSQL async driver
    "aiosqlite>=0.19.0",  # SQLite async driver
    "sqlglot>=20.0.0",
    "openai>=1.3.0",
    "openpyxl>=3.1.0",  # Excel export
    "python-multipart>=0.0.6",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.1.0",
    "mypy>=1.7.0",
    "ruff>=0.1.0",
    "httpx>=0.25.0",  # For testing FastAPI
]
```

### Frontend Dependencies

The `package.json` includes:

```json
{
  "name": "dataforge-frontend",
  "version": "0.1.0",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx",
    "test": "vitest"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "@refinedev/core": "^5.0.0",
    "@refinedev/react-router-v6": "^5.0.0",
    "@refinedev/simple-rest": "^5.0.0",
    "@refinedev/antd": "^5.0.0",
    "antd": "^5.11.0",
    "@monaco-editor/react": "^4.6.0",
    "axios": "^1.6.0",
    "tailwindcss": "^3.3.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@vitejs/plugin-react": "^4.2.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0",
    "vitest": "^1.0.0",
    "@testing-library/react": "^14.1.0",
    "eslint": "^8.55.0",
    "prettier": "^3.1.0"
  }
}
```

## Development Workflow

### Running Tests

**Backend:**
```bash
cd backend

# Run all tests
pytest

# Run with coverage
pytest --cov=src --cov-report=html

# Run specific test file
pytest tests/unit/test_query_parser.py -v

# Run integration tests (requires test database)
pytest tests/integration/ -v
```

**Frontend:**
```bash
cd frontend

# Run tests
npm test

# Run with coverage
npm run test:coverage

# Run specific test
npm test -- SQLEditor.test.tsx
```

### Type Checking

**Backend:**
```bash
cd backend

# Run mypy type checker
mypy src/ --strict
```

**Frontend:**
```bash
cd frontend

# Run TypeScript compiler (no emit)
npm run tsc -- --noEmit
```

### Linting and Formatting

**Backend:**
```bash
cd backend

# Check code with ruff
ruff check src/

# Auto-fix issues
ruff check src/ --fix

# Format code
ruff format src/
```

**Frontend:**
```bash
cd frontend

# Lint code
npm run lint

# Format with prettier
npm run format
```

## API Testing

### Using cURL

**List databases:**
```bash
curl http://localhost:8000/api/v1/dbs/
```

**Add database:**
```bash
curl -X PUT http://localhost:8000/api/v1/dbs/my_db \
  -H "Content-Type: application/json" \
  -d '{"url": "postgresql://user:pass@localhost:5432/mydb"}'
```

**Get database metadata:**
```bash
curl http://localhost:8000/api/v1/dbs/my_db
```

**Execute query:**
```bash
curl -X POST http://localhost:8000/api/v1/dbs/my_db/query \
  -H "Content-Type: application/json" \
  -d '{"sql": "SELECT * FROM users LIMIT 10"}'
```

**Natural language query:**
```bash
curl -X POST http://localhost:8000/api/v1/dbs/my_db/query/natural \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Show all users created in the last week"}'
```

### Using HTTPie

```bash
# Install httpie
pip install httpie

# List databases
http GET http://localhost:8000/api/v1/dbs/

# Add database
http PUT http://localhost:8000/api/v1/dbs/my_db \
  url="postgresql://user:pass@localhost:5432/mydb"

# Execute query
http POST http://localhost:8000/api/v1/dbs/my_db/query \
  sql="SELECT * FROM users LIMIT 10"
```

## Testing with Docker PostgreSQL

Quick test database setup:

```bash
# Start PostgreSQL in Docker
docker run --name dataforge-test-pg \
  -e POSTGRES_PASSWORD=testpass \
  -e POSTGRES_USER=testuser \
  -e POSTGRES_DB=testdb \
  -p 5432:5432 \
  -d postgres:16

# Create test table
docker exec -it dataforge-test-pg psql -U testuser -d testdb -c "
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (email) VALUES 
  ('alice@example.com'),
  ('bob@example.com'),
  ('charlie@example.com');
"

# Connection URL to use: postgresql://testuser:testpass@localhost:5432/testdb
```

## Validation Checklist

After setup, verify:

- [ ] Backend server starts without errors
- [ ] Frontend development server starts
- [ ] Can add database connection via UI
- [ ] Database metadata loads and displays tables/views
- [ ] Can execute SELECT query and see results
- [ ] Can't execute non-SELECT queries (error displayed)
- [ ] Query without LIMIT gets LIMIT 1000 added (warning shown)
- [ ] Natural language to SQL generates valid query
- [ ] Can export results to CSV format
- [ ] Can export results to Excel format
- [ ] Can export results to JSON format
- [ ] All API endpoints return camelCase field names
- [ ] Backend passes mypy type checking
- [ ] Frontend passes TypeScript compilation
- [ ] Tests pass for both backend and frontend

## Troubleshooting

### Backend won't start

**Issue**: `ModuleNotFoundError: No module named 'fastapi'`
**Solution**: Ensure virtual environment is activated and dependencies installed:
```bash
source .venv/bin/activate
uv pip install -e .
```

**Issue**: `KeyError: 'OPENAI_API_KEY'`
**Solution**: Set the environment variable:
```bash
export OPENAI_API_KEY="your-key"
```

### Frontend build errors

**Issue**: `Cannot find module '@refinedev/core'`
**Solution**: Reinstall dependencies:
```bash
rm -rf node_modules package-lock.json
npm install
```

### Database connection fails

**Issue**: `Connection refused` when adding database
**Solution**: 
1. Verify PostgreSQL is running: `pg_isready -h localhost -p 5432`
2. Check firewall allows port 5432
3. Verify credentials in connection URL

### SQL parsing errors

**Issue**: Valid SQL rejected as invalid syntax
**Solution**: Check sqlglot dialect setting - should be "postgres" for PostgreSQL databases

### Export downloads not working

**Issue**: Export URL returns 404
**Solution**: Check that export URLs haven't expired (15 minute TTL)

## Next Steps

After completing quickstart:

1. Review [data-model.md](./data-model.md) for full API schema
2. Check [contracts/openapi.yaml](./contracts/openapi.yaml) for complete API documentation
3. See [tasks.md](./tasks.md) for implementation task breakdown
4. Run full test suite to verify everything works

## Production Deployment

For production deployment:

1. Set `DATAFORGE_CORS_ORIGINS` to specific origins (not "*")
2. Use proper PostgreSQL connection pooling
3. Configure rate limiting for OpenAI API calls
4. Set up logging aggregation
5. Use environment-specific .env files
6. Build frontend for production: `npm run build`
7. Serve frontend static files via CDN or nginx
8. Run backend with production ASGI server (gunicorn + uvicorn workers)

Example production start:
```bash
gunicorn src.main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --access-logfile - \
  --error-logfile -
```

## Support

If you encounter issues:

1. Check this quickstart guide
2. Review error logs in terminal
3. Verify all environment variables are set
4. Ensure all dependencies are installed
5. Check that test database is accessible
