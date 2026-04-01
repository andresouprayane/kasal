# Kasal Developer Guide

**Audience**: Backend and Frontend Engineers

---

## Prerequisites

| Requirement | Minimum Version |
|-------------|----------------|
| Python | 3.10+ |
| Node.js | 18.0+ |
| uv (Python package manager) | Latest |
| npm | 9.0+ |
| Git | 2.x |

---

## Local Development Setup

### 1. Clone the Repository

```bash
git clone https://github.com/databrickslabs/kasal
cd kasal
```

### 2. Start the Backend

```bash
cd src/backend
./run.sh
```

`run.sh` uses `uv` to sync dependencies and starts uvicorn with `--reload`. The backend auto-restarts on any Python file change.

The API will be available at `http://localhost:8000`.  
Interactive docs: `http://localhost:8000/api-docs`

### 3. Start the Frontend

```bash
cd src/frontend
npm install
npm start
```

The frontend starts with hot module replacement (HMR) at `http://localhost:3000`.

> **Note**: Do not restart the backend or frontend services manually. They auto-reload on file changes.

---

## Project Layout

```
src/
├── backend/
│   ├── src/
│   │   ├── api/            # FastAPI routers (42 files, one per domain)
│   │   ├── services/       # Business logic services (57+)
│   │   ├── repositories/   # Data access objects (23+)
│   │   ├── models/         # SQLAlchemy ORM models (32)
│   │   ├── schemas/        # Pydantic V2 schemas
│   │   ├── engines/
│   │   │   └── crewai/     # CrewAI execution engine
│   │   ├── core/           # Auth, permissions, exceptions
│   │   ├── config/         # Settings and database setup
│   │   ├── db/             # Session factory, base model
│   │   ├── middleware/      # Group context, logging
│   │   └── seeds/          # Startup seed data
│   ├── tests/              # pytest test suite
│   ├── migrations/         # Alembic migrations
│   └── pyproject.toml      # Python dependencies
└── frontend/
    └── src/
        ├── components/     # React UI components
        ├── store/          # Redux Toolkit slices (31)
        ├── api/            # Axios service wrappers
        ├── hooks/          # Custom React hooks
        ├── types/          # TypeScript types
        └── utils/          # Shared utilities
```

---

## Architecture Principles

Every feature in Kasal follows the same four-layer pattern:

```
API Router  →  Service  →  Repository  →  Database
```

### Adding a New Feature — Checklist

1. **Model** (`models/`): Define the SQLAlchemy ORM class
2. **Schema** (`schemas/`): Define Pydantic request/response models
3. **Repository** (`repositories/`): Implement data access methods
4. **Service** (`services/`): Implement business logic; call the repository
5. **Router** (`api/`): Define FastAPI endpoints; call the service
6. **Migration** (`migrations/`): Generate and apply the Alembic migration
7. **Frontend API client** (`frontend/src/api/`): Add the Axios service wrapper
8. **Redux slice** (`frontend/src/store/`): Add state management if needed
9. **React component** (`frontend/src/components/`): Build the UI

---

## Backend Conventions

### Async Everywhere

All service and repository methods that perform I/O must be `async`:

```python
# Correct
async def get_agent(self, agent_id: str) -> Agent:
    return await self.repository.get_by_id(agent_id)

# Wrong - never do this
def get_agent(self, agent_id: str) -> Agent:
    return self.repository.get_by_id(agent_id)
```

### Dependency Injection

Use FastAPI's `Depends()` for all service/repository injection:

```python
# api/agents_router.py
@router.get("/{agent_id}", response_model=AgentResponse)
async def get_agent(
    agent_id: str,
    group_id: str = Depends(get_current_group),
    service: AgentService = Depends(get_agent_service),
):
    return await service.get(agent_id, group_id)
```

### Multi-Tenant Scoping

Every create/read/update/delete operation must pass `group_id`:

```python
# Always scope queries to the current group
async def list_agents(self, group_id: str) -> list[Agent]:
    result = await self.session.execute(
        select(Agent).where(Agent.group_id == group_id)
    )
    return result.scalars().all()
```

### Error Handling

Use HTTP exceptions at the router layer; raise domain exceptions from services:

```python
# service
async def get_agent(self, agent_id: str, group_id: str) -> Agent:
    agent = await self.repository.get_by_id(agent_id, group_id)
    if not agent:
        raise AgentNotFoundError(agent_id)
    return agent

# router (catches domain errors and maps to HTTP)
@router.get("/{agent_id}")
async def get_agent(agent_id: str, ...):
    try:
        return await service.get_agent(agent_id, group_id)
    except AgentNotFoundError:
        raise HTTPException(status_code=404, detail="Agent not found")
```

### No Real URLs in Code

Never hardcode real endpoints or credentials. Use environment variables or placeholder strings:

```python
# Correct
base_url = settings.DATABRICKS_HOST  # from env

# Wrong
base_url = "https://adb-1234567890.1.azuredatabricks.net"
```

---

## Database Migrations

Kasal uses Alembic for schema migrations.

```bash
cd src/backend

# Generate a new migration from model changes
uv run alembic revision --autogenerate -m "add schedule table"

# Apply pending migrations
uv run alembic upgrade head

# Roll back one migration
uv run alembic downgrade -1
```

Migration files are in `migrations/versions/`. Always review auto-generated migrations before committing — Alembic may miss index changes or complex constraints.

---

## Running Tests

### Backend Tests

```bash
cd src/backend

# Run all tests
uv run python run_tests.py

# Run a specific test file
uv run pytest tests/unit/test_agent_service.py -v

# Run with coverage
uv run pytest tests/ --cov=src --cov-report=html
```

Target: **80% minimum coverage** for all service and repository code.

### Test File Location

All test scripts and temporary files go in `/tmp/`. Do not create test files in the project directory:

```bash
# Correct
/tmp/test_my_feature.py

# Wrong
src/backend/test_my_feature.py
```

### Frontend Tests

```bash
cd src/frontend

# Run all tests
npm test

# Run with coverage
npm test -- --coverage
```

---

## Adding a New API Endpoint — End-to-End Example

Suppose you want to add `GET /api/v1/tools/{tool_id}/config`.

### Step 1: Model (if new table needed)

If the data is already in an existing model, skip this step.

### Step 2: Schema

```python
# src/backend/src/schemas/tool.py
class ToolConfigResponse(BaseModel):
    tool_id: str
    config: dict
    updated_at: datetime
```

### Step 3: Repository

```python
# src/backend/src/repositories/tool_repository.py
async def get_config(self, tool_id: str, group_id: str) -> dict:
    tool = await self.get_by_id(tool_id, group_id)
    return tool.config if tool else {}
```

### Step 4: Service

```python
# src/backend/src/services/tool_service.py
async def get_tool_config(self, tool_id: str, group_id: str) -> ToolConfigResponse:
    config = await self.repository.get_config(tool_id, group_id)
    return ToolConfigResponse(tool_id=tool_id, config=config, updated_at=datetime.utcnow())
```

### Step 5: Router

```python
# src/backend/src/api/tools_router.py
@router.get("/{tool_id}/config", response_model=ToolConfigResponse)
async def get_tool_config(
    tool_id: str,
    group_id: str = Depends(get_current_group),
    service: ToolService = Depends(get_tool_service),
):
    return await service.get_tool_config(tool_id, group_id)
```

### Step 6: Frontend API Client

```typescript
// src/frontend/src/api/ToolService.ts
export const getToolConfig = async (toolId: string): Promise<ToolConfigResponse> => {
  const response = await axiosInstance.get<ToolConfigResponse>(`/tools/${toolId}/config`);
  return response.data;
};
```

---

## Common Development Tasks

### Check Backend Service Status

```bash
ps aux | grep uvicorn
```

### Check Frontend Service Status

```bash
ps aux | grep "npm start"
```

### Database Inspection (SQLite Development)

```bash
cd src/backend
uv run python -c "
import asyncio
from src.db.session import get_db
from src.models.agent import Agent
from sqlalchemy import select

async def main():
    async with get_db() as session:
        result = await session.execute(select(Agent).limit(5))
        for agent in result.scalars().all():
            print(agent.id, agent.name)

asyncio.run(main())
"
```

### Seed the Database

The database is auto-seeded at startup if `AUTO_SEED_DATABASE=true`. To manually trigger:

```bash
cd src/backend
uv run python -c "from src.seeds.runner import run_seeds; import asyncio; asyncio.run(run_seeds())"
```

### Build Frontend Static Assets

```bash
python src/build.py
```

---

## Key Files Quick Reference

| File | Purpose |
|------|---------|
| `src/backend/src/config/settings.py` | All environment-driven settings |
| `src/backend/src/main.py` | FastAPI app factory, startup/shutdown handlers |
| `src/backend/src/db/session.py` | Async session factory |
| `src/backend/src/core/permissions.py` | RBAC enforcement |
| `src/backend/src/engines/crewai/crew_preparation.py` | Build CrewAI objects from config |
| `src/backend/src/engines/crewai/process_crew_executor.py` | Subprocess-based crew execution |
| `src/backend/src/engines/crewai/tools/tool_factory.py` | Dynamic tool loading |
| `src/frontend/src/store/` | All Redux state management |
| `src/frontend/src/api/` | All backend API clients |
| `src/frontend/src/components/WorkflowDesigner/` | Main visual canvas |

---

## Linting and Code Quality

### Backend

```bash
cd src/backend

# Type checking
uv run mypy src/

# Linting
uv run ruff check src/

# Format
uv run ruff format src/
```

### Frontend

```bash
cd src/frontend

# Type checking
npx tsc --noEmit

# Lint
npm run lint
```

Never commit without passing linting and type checks.

---

## Debugging Tips

### Trace a Feature End-to-End

1. Find the router in `src/backend/src/api/` by searching for the URL path
2. Find the service call — the router calls `service.method()`
3. Find the repository call inside the service
4. Find the model definition in `models/`

### Enable Debug Logging

Set `LOG_LEVEL=DEBUG` in your environment before starting the backend:

```bash
LOG_LEVEL=DEBUG ./run.sh
```

### Inspect Execution Details

Execution logs, task statuses, and traces are queryable via:
- `GET /api/v1/executions/{job_id}/logs`
- `GET /api/v1/executions/{job_id}/trace`
- `GET /api/v1/executions/{job_id}/tasks`

---

## Frontend Development Tips

### Redux DevTools

Install the Redux DevTools browser extension to inspect all state changes in real time.

### Mock Data for UI Development

Use the Axios mock adapter in test files to simulate API responses without a running backend.

### ReactFlow Customization

Custom node and edge types are defined in `components/WorkflowDesigner/`. Register new node types in the `nodeTypes` map passed to the `ReactFlow` component.
