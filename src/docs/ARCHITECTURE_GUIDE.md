# Kasal Solution Architecture

**Audience**: Solution Architects, Technical Leads, Senior Engineers

---

## Overview

Kasal is an enterprise AI agent workflow orchestration platform. It follows **Clean Architecture** principles with strict unidirectional layer dependencies, async-first I/O, and group-based multi-tenancy.

```
┌─────────────────────────────────────────────────────┐
│                  Frontend (React)                    │
│         Visual Designer  |  Monitoring  |  Config    │
└───────────────────────────┬─────────────────────────┘
                            │ REST / WebSocket
┌───────────────────────────▼─────────────────────────┐
│                  API Layer (FastAPI)                  │
│          Routers  |  Schemas  |  Auth Middleware      │
└───────────────────────────┬─────────────────────────┘
                            │ Dependency Injection
┌───────────────────────────▼─────────────────────────┐
│               Service Layer (Business Logic)          │
│     Execution  |  Agent  |  Crew  |  Scheduler       │
└───────────────────────────┬─────────────────────────┘
                            │ Unit of Work / Repository Pattern
┌───────────────────────────▼─────────────────────────┐
│              Repository Layer (Data Access)           │
│       SQLAlchemy  |  Vector Search  |  External APIs  │
└───────────────────────────┬─────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────┐
│                     Storage                          │
│     PostgreSQL / SQLite  |  Databricks Vector Search  │
└─────────────────────────────────────────────────────┘
```

---

## Core Architectural Patterns

### 1. Clean Architecture (Layered)

Each layer has a single responsibility and communicates only with the adjacent layer below:

- **API Layer**: HTTP request parsing, input validation, auth enforcement, response serialization
- **Service Layer**: Business logic, orchestration, cross-cutting concerns
- **Repository Layer**: All data access — SQL queries, external API calls, vector search
- **Storage**: Persistent state (database, vector indices)

**Rule**: Services call repositories. Repositories never call services. API routes never access repositories directly.

### 2. Repository Pattern

Every data entity has a dedicated repository class that encapsulates all database interactions:

```python
class AgentRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, agent_id: str, group_id: str) -> Agent | None:
        result = await self.session.execute(
            select(Agent).where(Agent.id == agent_id, Agent.group_id == group_id)
        )
        return result.scalar_one_or_none()
```

Repositories accept an `AsyncSession` injected at construction. They never manage transactions themselves.

### 3. Unit of Work Pattern

Transactions span multiple repository operations via a shared `AsyncSession`. Services receive an already-open session from the dependency injection system, ensuring atomicity:

```python
async def create_crew_with_agents(crew_data, session: AsyncSession):
    agent_repo = AgentRepository(session)
    crew_repo = CrewRepository(session)
    # Both share the same session; commit/rollback applies to both
    agents = await agent_repo.create_many(crew_data.agents)
    crew = await crew_repo.create(crew_data, agent_ids=[a.id for a in agents])
    await session.commit()
```

### 4. Async-First

All I/O operations use `async/await`. This includes:
- Every database query (SQLAlchemy 2.0 with `asyncpg` or `aiosqlite`)
- All external HTTP calls (httpx, databricks-sdk async client)
- All service methods that perform any I/O

Synchronous blocking calls are not permitted in request handlers.

### 5. Dependency Injection

FastAPI's `Depends()` mechanism injects services and repositories:

```python
@router.get("/agents")
async def list_agents(
    group_id: str = Depends(get_current_group),
    service: AgentService = Depends(get_agent_service),
):
    return await service.list(group_id)
```

---

## System Components

### Backend (`src/backend/src/`)

| Directory | Contents |
|-----------|---------|
| `api/` | 42 FastAPI routers, one per domain |
| `services/` | 57+ business logic services |
| `repositories/` | 23+ data access objects |
| `models/` | 32 SQLAlchemy ORM models |
| `schemas/` | Pydantic V2 request/response schemas |
| `engines/` | CrewAI execution engine (crew_preparation, flow_preparation, execution_runner) |
| `core/` | Auth, permissions, exception handling |
| `config/` | Settings, database configuration |
| `db/` | Session factory, declarative base |
| `middleware/` | Group context extraction, request logging |
| `seeds/` | Database seed data (model configs, tool defaults) |
| `utils/` | Shared helpers |

### Frontend (`src/frontend/src/`)

| Directory | Contents |
|-----------|---------|
| `components/` | React UI components in 15+ feature directories |
| `store/` | 31 Redux Toolkit slices |
| `api/` | Axios service wrappers for each backend domain |
| `hooks/` | Custom React hooks |
| `types/` | TypeScript type definitions |
| `utils/` | Shared utilities |
| `theme/` | Material-UI theme configuration |

---

## Execution Architecture

### CrewAI Engine (`src/backend/src/engines/crewai/`)

Crew execution is the platform's core capability. The full pipeline:

```
API Request (POST /executions)
        │
        ▼
ExecutionService
  - Creates ExecutionHistory record (status: pending)
  - Dispatches to DispatcherService
        │
        ▼
DispatcherService
  - Determines: crew execution or flow execution
  - Creates async background task
        │
        ▼
ProcessCrewExecutor (separate OS process)
  - Isolation: runs in subprocess to prevent memory leaks
  - CrewPreparation: builds Agent/Task/Crew objects from config
  - Resolves tools via ToolFactory
  - Configures memory backends
  - Calls crew.kickoff() or flow.kickoff()
        │
        ▼
CrewAI Framework
  - Sequential or hierarchical process
  - Agent LLM calls via LiteLLM
  - Tool invocations
  - Callbacks → ExecutionLogsService
        │
        ▼
Results stored in ExecutionHistory
Logs stored in ExecutionLogs
Traces stored in ExecutionTrace
```

### Process Isolation

Crew execution runs in a **separate OS process** via `ProcessCrewExecutor`. Benefits:
- Prevents memory leaks from long-running AI agents
- Provides crash isolation (subprocess failure does not kill the API server)
- Enables graceful stop via `SIGTERM` to the subprocess

### Graceful Stop

When a user requests to stop an execution:
1. `ExecutionService.stop_execution()` sets `stop_requested = True` in the database
2. The polling watchdog in `ProcessCrewExecutor` detects this flag
3. Sends `SIGTERM` to the crew subprocess
4. Updates execution status to `stopped`

---

## Data Model Overview

### Core Entities

| Entity | Purpose |
|--------|---------|
| `Agent` | AI agent with role, goal, backstory, LLM config, tools, memory settings |
| `Task` | Unit of work assigned to an agent with expected output |
| `Crew` | Collection of agents and tasks with workflow topology (nodes, edges) |
| `ExecutionHistory` | Lifecycle record for a run (status, inputs, result, timestamps) |
| `TaskStatus` | Per-task status tracking within an execution |
| `ExecutionTrace` | Detailed event trace of an execution |
| `Group` | Tenant workspace with user assignments |
| `User` | Platform user with role and group memberships |
| `MemoryBackend` | Configuration for agent memory (type, connection details, embedder) |
| `Tool` | Custom tool registry entry |
| `Schedule` | Cron-based execution schedule |

### Multi-Tenancy

Every tenant-scoped entity has a `group_id` field. The `GroupContext` middleware extracts the tenant from:
1. `X-Databricks-User` or `X-Forwarded-User` header (Databricks Apps deployment)
2. JWT claims (standalone deployment)

All repository queries filter by `group_id`. Services receive it via `Depends(get_current_group)`.

---

## Security Architecture

### Authentication Flow

```
Client
  │
  ├── JWT Login ─────────────► POST /auth/login
  │                                    │
  │                            Validate credentials
  │                            Issue access_token (8 days)
  │                            Issue refresh_token (httpOnly cookie)
  │
  ├── API Requests ──────────► Authorization: Bearer <access_token>
  │                                    │
  │                            JWT middleware validates token
  │                            Injects user into request state
  │
  └── Databricks OAuth ──────► POST /auth/oauth/{provider}/authorize
                                       │
                                OAuth 2.0 redirect → callback
                                Exchange code for tokens
                                Create/update platform user
```

### Role-Based Access Control

Four roles in ascending privilege:

| Role | Capabilities |
|------|-------------|
| **Operator** | View executions, trigger runs, view logs |
| **Editor** | Create/edit agents, tasks, crews; all Operator capabilities |
| **Workspace Admin** | Manage users and groups in their workspace; all Editor capabilities |
| **System Admin** | Full platform access including system configuration |

Role enforcement is implemented in `core/permissions.py` via a `check_permission()` decorator applied to service methods.

---

## Memory Architecture

Kasal supports configurable memory backends per crew:

| Backend | Storage | Use Case |
|---------|---------|---------|
| **In-Memory** | Process heap | Development, short-lived runs |
| **SQLite** | Local file | Single-node persistent memory |
| **Databricks Vector Search** | Delta Lake + Vector Index | Production semantic search |

Memory types within each backend:
- **Short-term**: Recent conversation context (last N turns)
- **Long-term**: Persisted facts across runs
- **Entity**: Named entity extraction and tracking (uses a secondary LLM call)

---

## Tool Architecture

### Tool Registry

Tools are registered as `Tool` records in the database and loaded dynamically at execution time by `ToolFactory` in `engines/crewai/tools/`.

### Native Tools

| Tool | Description |
|------|------------|
| `DatabricksJobsTool` | Trigger and monitor Databricks Jobs |
| `GenieTool` | Natural language queries via Databricks Genie |
| `PerplexityTool` | Web research and search |
| `DatabricksKnowledgeSearchTool` | Search Databricks Vector Search indices |

### MCP (Model Context Protocol) Tools

The `MCPService` manages connections to external MCP tool servers. Tools are discovered dynamically and exposed to agents as standard CrewAI-compatible tools. MCP settings are stored in `mcp_settings` table and can be configured per workspace.

---

## Observability

### Execution Logs

All agent and task events are streamed to `ExecutionLogs` via callbacks registered on the CrewAI crew object. The frontend polls `GET /executions/{job_id}/logs` to display real-time output.

### Execution Traces

`ExecutionTrace` records structured event data (agent thought, tool call, tool result, task output) enabling post-run inspection and debugging.

### MLflow Integration

For experiment tracking and evaluation:
- `MLflowService` creates experiment runs associated with executions
- Task outputs are logged as artifacts
- Evaluation metrics stored for model/prompt comparison

### Structured Logging

The backend outputs JSON-structured logs. Log level is controlled by the `LOG_LEVEL` environment variable (default: `INFO`).

---

## Scheduling Architecture

`SchedulerService` (APScheduler) manages cron-based execution:

1. Schedules are persisted as `Schedule` records
2. At startup, all active schedules are loaded into APScheduler's in-process store
3. When a cron fires, `ExecutionService.run_execution()` is called in a background task
4. Schedule CRUD operations update both the database and the live scheduler instance

---

## Frontend Architecture

### State Management

31 Redux Toolkit slices:

| Slice | Responsibility |
|-------|--------------|
| `workflow` | Canvas nodes and edges (ReactFlow state) |
| `crewExecution` | Active execution tracking and control |
| `runStatus` | Execution status polling |
| `agent`, `crew`, `task` | Entity CRUD state |
| `auth` | JWT tokens and user session |
| `groups`, `user` | Multi-tenant context |
| `permissions` | Role-based UI permission state |
| `tabManager`, `uiLayout` | UI state (open tabs, panel sizes) |

### Component Architecture

```
App
├── WorkflowDesigner          # Main visual workflow page
│   ├── CrewCanvas            # ReactFlow graph (agents/tasks as nodes)
│   ├── LeftSidebar           # Agent/task palette
│   ├── RightSidebar          # Property inspector
│   ├── TabBar                # Multi-crew tab management
│   └── WorkflowToolbar       # Run/Stop/Save controls
├── ExecutionHistory          # Past run browser with logs/traces
├── Tools                     # Tool registration and management
├── Configuration             # Engine, model, memory, MCP settings
└── Documentation             # In-app markdown docs viewer
```

---

## Integration Patterns

### Databricks Apps Deployment

When deployed as a Databricks App:
- `X-Forwarded-User` and `X-Databricks-User` headers carry user identity
- `GroupContext` middleware extracts group/workspace from these headers
- OAuth flows use the Databricks OAuth provider in `manifest.yaml`
- Secrets accessed via Databricks Secrets API (no `.env` files in production)

### LLM Provider Abstraction

LiteLLM provides a unified interface for 100+ LLM providers. Agents specify their LLM as a string:
- `"databricks/databricks-meta-llama-3-1-70b-instruct"`
- `"openai/gpt-4o"`
- `"anthropic/claude-3-5-sonnet-20241022"`

LiteLLM handles provider-specific authentication and request formatting, allowing model swaps without code changes.

---

## Design Decisions

| Decision | Rationale |
|----------|----------|
| Process isolation for crew execution | Prevents a crashed/hung crew from destabilizing the API server |
| SQLite for development | Zero-dependency local setup; PostgreSQL for production |
| uv for Python packaging | Faster than pip, reproducible lockfiles, better dependency resolution |
| ReactFlow for canvas | Best-in-class graph visualization for React workflow UIs |
| LiteLLM over direct provider SDKs | Single interface for all LLMs; allows model swapping at runtime |
| Group-based multi-tenancy | Simpler than schema-per-tenant; sufficient for enterprise deployments |
| Pydantic V2 schemas | Faster validation, better error messages, strict mode support |
