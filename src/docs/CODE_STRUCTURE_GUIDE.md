# Kasal Code Structure Guide

A fast, skimmable map of the repository to help you find the right place quickly.

---

## Repository Layout

```
kasal/
├── src/
│   ├── backend/                  # Python FastAPI backend
│   │   ├── src/                  # Application source code
│   │   ├── tests/                # pytest test suite
│   │   ├── migrations/           # Alembic database migrations
│   │   │   └── versions/         # Individual migration files
│   │   ├── pyproject.toml        # Python dependencies (uv)
│   │   └── run.sh                # Development server startup
│   ├── frontend/                 # React TypeScript frontend
│   │   ├── src/                  # Application source code
│   │   ├── public/               # Static assets and HTML template
│   │   └── package.json          # npm dependencies
│   ├── frontend_static/          # Production-built frontend assets
│   └── docs/                     # This documentation
├── README.md                     # Project overview
├── CONTRIBUTING.md               # Contribution guidelines
├── CLAUDE.md                     # AI development guidelines
└── manifest.yaml                 # Databricks Apps metadata
```

---

## Backend Code Map (`src/backend/src/`)

### `api/` — FastAPI Routers (42 files)

Each file contains one router for one domain. All routers are registered in `main.py`.

| File | Domain |
|------|--------|
| `auth_router.py` | Login, logout, token refresh, Databricks OAuth |
| `agents_router.py` | Agent CRUD |
| `tasks_router.py` | Task CRUD |
| `crews_router.py` | Crew CRUD |
| `executions_router.py` | Start, stop, list executions |
| `execution_logs_router.py` | Real-time and historical execution logs |
| `execution_trace_router.py` | Structured event traces |
| `execution_history_router.py` | Past run history |
| `tools_router.py` | Tool registry management |
| `models_router.py` | LLM model configuration |
| `engine_config_router.py` | CrewAI engine settings |
| `scheduler_router.py` | Cron schedule management |
| `memory_backend_router.py` | Memory backend configuration |
| `mcp_router.py` | MCP server settings |
| `group_router.py` | Workspace/group management |
| `databricks_*_router.py` | Databricks connectors (secrets, jobs, genie, volumes, index) |
| `user_router.py` | User profile management |
| `pipeline_router.py` | Pipeline execution |
| `flow_router.py` | Multi-crew flow management |

### `services/` — Business Logic (57+ files)

Services orchestrate repositories and implement all business rules.

| File | Responsibility |
|------|---------------|
| `execution_service.py` | Execution lifecycle management (create, stop, status) |
| `crewai_execution_service.py` | CrewAI-specific execution coordinator |
| `process_crew_executor.py` | Subprocess-based crew runner (117KB — core engine) |
| `dispatcher_service.py` | Routes executions to appropriate executor |
| `agent_service.py` | Agent CRUD and validation |
| `task_service.py` | Task CRUD |
| `crew_service.py` | Crew CRUD and topology validation |
| `agent_generation_service.py` | AI-assisted agent config generation |
| `crew_generation_service.py` | AI-assisted crew generation from a prompt |
| `execution_history_service.py` | Historical run queries and aggregation |
| `execution_status_service.py` | Status update and notification |
| `execution_logs_service.py` | Log streaming and persistence |
| `execution_trace_service.py` | Trace event capture |
| `scheduler_service.py` | APScheduler integration for cron jobs |
| `execution_cleanup_service.py` | Housekeeping for old executions |
| `tool_service.py` | Tool registry operations |
| `mcp_service.py` | MCP server connection and tool discovery |
| `databricks_service.py` | Databricks API integration |
| `databricks_secrets_service.py` | Databricks Secrets management |
| `databricks_knowledge_service.py` | Knowledge base management |
| `databricks_index_service.py` | Vector Search index operations |
| `mlflow_service.py` | MLflow experiment and run management |
| `mlflow_evaluation_runner.py` | Automated evaluation runs |
| `dspy_optimization_service.py` | DSPy prompt optimization |
| `memory_backend_service.py` | Memory backend CRUD |
| `knowledge_embedding_service.py` | Document embedding and indexing |
| `knowledge_search_service.py` | Semantic search queries |
| `template_generation_service.py` | Example/template workflow generation |
| `documentation_embedding_service.py` | Documentation search indexing |

### `repositories/` — Data Access (23+ files)

Pure data access — no business logic here.

| File | Entity |
|------|--------|
| `agent_repository.py` | Agent CRUD with group scoping |
| `task_repository.py` | Task CRUD |
| `crew_repository.py` | Crew CRUD |
| `execution_repository.py` | Execution lifecycle queries |
| `execution_history_repository.py` | Historical execution queries |
| `execution_logs_repository.py` | Log read/write |
| `execution_trace_repository.py` | Trace event persistence |
| `tool_repository.py` | Tool registry queries |
| `databricks_vector_index_repository.py` | Vector index CRUD (58KB) |
| `databricks_volume_repository.py` | Volume management |
| `mlflow_config_repository.py` | MLflow configuration |
| `mcp_settings_repository.py` | MCP server settings |
| `memory_backend_repository.py` | Memory backend configuration |
| `schedule_repository.py` | Cron schedule persistence |
| `user_repository.py` | User management |
| `group_repository.py` | Group/workspace management |

### `models/` — SQLAlchemy ORM (32 files)

Each file defines one database table as a SQLAlchemy declarative model.

| File | Table | Key Fields |
|------|-------|-----------|
| `agent.py` | `agents` | role, goal, backstory, llm, tools, memory, group_id |
| `task.py` | `tasks` | description, expected_output, agent_id, tools, async_execution, group_id |
| `crew.py` | `crews` | name, agent_ids, task_ids, nodes, edges, group_id |
| `execution_history.py` | `execution_history` | job_id, status, inputs, result, started_at, completed_at, group_id |
| `task_status.py` | `task_statuses` | job_id, task_id, agent_name, status |
| `error_trace.py` | `error_traces` | run_id, task_key, error_type, error_message |
| `execution_trace.py` | `execution_traces` | job_id, event_type, data |
| `user.py` | `users` | email, hashed_password, role, group_id |
| `group.py` | `groups` | name, description |
| `tool.py` | `tools` | name, description, code, config, group_id |
| `memory_backend.py` | `memory_backends` | type, connection_config, embedder_config |
| `schedule.py` | `schedules` | cron_expression, crew_id, inputs, is_active |
| `flow.py` | `flows` | name, crew_ids, config, group_id |
| `schema.py` | `schemas` | name, fields, group_id |
| `dspy_config.py` | `dspy_configs` | optimizer, metric, model |

### `schemas/` — Pydantic V2 Schemas

Mirror the models for API contracts. Each model has corresponding `Create`, `Update`, and `Response` schema classes.

### `engines/crewai/` — AI Execution Engine

The heart of Kasal — translates database configurations into running AI crews.

| File/Directory | Purpose |
|----------------|---------|
| `crew_preparation.py` | Build CrewAI Agent/Task/Crew objects from DB config |
| `flow_preparation.py` | Build multi-crew flows for flow execution |
| `execution_runner.py` | Main execution loop with asyncio |
| `process_crew_executor.py` | OS-process isolation for crew execution (117KB) |
| `config_adapter.py` | Normalize raw config for CrewAI API |
| `crew_logger.py` | Centralized crew-level logging |
| `trace_management.py` | Capture and persist execution traces |
| `tools/tool_factory.py` | Dynamic tool loading and instantiation (62KB) |
| `tools/databricks_jobs_tool.py` | Native Databricks Jobs integration |
| `tools/genie_tool.py` | Databricks Genie NL query tool |
| `tools/perplexity_tool.py` | Web research tool |
| `tools/databricks_knowledge_search_tool.py` | Vector Search tool |
| `memory/` | Memory backend factory and implementations |
| `callbacks/logging_callbacks.py` | CrewAI event listeners for log streaming |
| `guardrails/` | Task-level safety constraints |
| `config/` | Embedder, manager, and crew configuration builders |

### `core/` — Cross-Cutting Concerns

| File | Purpose |
|------|---------|
| `permissions.py` | RBAC enforcement — `check_permission()` decorator |
| `auth.py` | JWT creation, validation, refresh token logic |
| `exceptions.py` | Domain exception classes |
| `security.py` | Password hashing utilities |

### `config/` — Application Configuration

| File | Purpose |
|------|---------|
| `settings.py` | Pydantic Settings — all environment variables |
| `database.py` | SQLAlchemy engine and session factory configuration |

### `middleware/` — Request Middleware

| File | Purpose |
|------|---------|
| `group_context.py` | Extract group_id from Databricks headers or JWT |
| `logging_middleware.py` | Request/response logging |

### `seeds/` — Startup Data

Seeds run at application startup when `AUTO_SEED_DATABASE=true`.

| File | Seeds |
|------|-------|
| `model_configs.py` | Default LLM model configurations |
| `tool_configs.py` | Built-in tool definitions |
| `engine_configs.py` | Default CrewAI engine settings |

---

## Frontend Code Map (`src/frontend/src/`)

### `components/` — React UI Components

| Directory | Contents |
|-----------|---------|
| `WorkflowDesigner/` | Main canvas — CrewCanvas, FlowCanvas, sidebars, toolbar |
| `Agents/` | Agent create/edit forms and list |
| `Tasks/` | Task create/edit forms and list |
| `Crews/` | Crew composition UI |
| `Jobs/` | ExecutionHistory, ExecutionLogs, ExecutionTrace viewer |
| `Tools/` | Tool registration and management |
| `Configuration/` | Engine, model, memory, MCP settings forms |
| `MemoryBackend/` | Memory backend configuration |
| `Documentation/` | In-app markdown documentation viewer |
| `Common/` | Shared buttons, dialogs, badges, status indicators |
| `Examples/` | Tutorial and example workflow browser |
| `Layout/` | AppBar, drawer, navigation |
| `Auth/` | Login, OAuth callback pages |
| `Scheduling/` | Schedule creation and management UI |
| `DSPy/` | DSPy optimization configuration |

### `store/` — Redux Toolkit Slices (31 files)

| Slice | State Managed |
|-------|--------------|
| `agent.ts` | Agent list and selected agent |
| `crew.ts` | Crew list and selected crew |
| `task.ts` | Task list and selected task |
| `workflow.ts` | ReactFlow nodes and edges |
| `crewExecution.ts` | Active execution id, status, controls |
| `runStatus.ts` | Execution status polling |
| `runResult.ts` | Execution result and output |
| `auth.ts` | User session, JWT token |
| `groups.ts` | Available groups and current group |
| `user.ts` | Current user profile |
| `permissions.ts` | Resolved permissions for current user/role |
| `tabManager.ts` | Open designer tabs |
| `uiLayout.ts` | Panel sizes, sidebar state |
| `shortcuts.ts` | Keyboard shortcut bindings |
| `memoryBackend.ts` | Memory backend configuration |
| `tools.ts` | Tool registry state |
| `models.ts` | Available LLM models |
| `engineConfig.ts` | CrewAI engine settings |
| `schedule.ts` | Schedule list and configuration |

### `api/` — Backend Service Wrappers

Each file wraps one backend domain with typed Axios calls.

| File | Backend Domain |
|------|---------------|
| `AgentService.ts` | `/api/v1/agents` |
| `TaskService.ts` | `/api/v1/tasks` |
| `CrewService.ts` | `/api/v1/crews` |
| `ExecutionService.ts` | `/api/v1/executions` |
| `ToolService.ts` | `/api/v1/tools` |
| `MemoryBackendService.ts` | `/api/v1/memory-backends` |
| `ModelConfigService.ts` | `/api/v1/models` |
| `DatabricksService.ts` | `/api/v1/databricks/*` |
| `LanguageService.ts` | `/api/v1/languages` |
| `DatabaseManagementService.ts` | `/api/v1/database` |

### `hooks/` — Custom React Hooks

| Hook | Purpose |
|------|---------|
| `usePermissions.ts` | Check role-based permissions with caching |
| `useWorkflow*.ts` | Canvas state and interaction helpers |
| `useAgent*.ts` | Agent CRUD and selection helpers |
| `useCrew*.ts` | Crew state helpers |
| `useExecution*.ts` | Execution lifecycle hooks |

### `types/` — TypeScript Definitions

Mirror backend Pydantic schemas as TypeScript interfaces. Located alongside their consuming components or in a shared `types/` directory.

---

## Naming Conventions

### Backend

| Artifact | Convention | Example |
|----------|-----------|---------|
| Files | `snake_case.py` | `agent_service.py` |
| Classes | `PascalCase` | `AgentService` |
| Methods | `snake_case` | `get_agent_by_id` |
| Variables | `snake_case` | `group_id` |
| DB columns | `snake_case` | `created_at` |
| Env vars | `UPPER_SNAKE_CASE` | `DATABASE_TYPE` |

### Frontend

| Artifact | Convention | Example |
|----------|-----------|---------|
| Files | `PascalCase.tsx` for components | `AgentForm.tsx` |
| Files | `camelCase.ts` for utilities | `agentService.ts` |
| Components | `PascalCase` | `AgentForm` |
| Functions | `camelCase` | `getAgentById` |
| Types/Interfaces | `PascalCase` with descriptive suffix | `AgentFormProps`, `AgentResponse` |
| Redux slices | `camelCase` | `agentSlice` |

---

## Finding Things Quickly

| Question | Where to Look |
|----------|--------------|
| How does endpoint X work? | `api/` → find the router file → trace the service call |
| Where is business logic for feature Y? | `services/y_service.py` |
| What database columns exist for entity Z? | `models/z.py` |
| What does the API request/response look like? | `schemas/z.py` |
| Where is the UI for feature Y? | `components/Y/` |
| What state does component X use? | `store/x.ts` |
| How does crew execution work? | `engines/crewai/process_crew_executor.py` |
| How are tools loaded? | `engines/crewai/tools/tool_factory.py` |
