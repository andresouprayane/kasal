# Kasal Data Models

This document describes the Kasal data model at three levels of abstraction:

- **Conceptual** — business concepts and how they relate, without technical detail
- **Logical** — entities with key attributes and typed relationships, technology-agnostic
- **Physical** — actual database tables, column types, primary keys, and foreign keys

---

## 1. Conceptual Data Model

Focuses on the core business domain. Shows *what* the system tracks, not *how*.

```mermaid
erDiagram
    USER {
        string identity
        string role
    }
    GROUP {
        string name
        string status
    }
    AGENT {
        string role
        string goal
        string llm_model
    }
    TASK {
        string description
        string expected_output
    }
    CREW {
        string name
        json workflow_topology
    }
    FLOW {
        string name
        json multi_crew_config
    }
    TOOL {
        string title
        json config
    }
    SCHEDULE {
        string cron_expression
        bool is_active
    }
    MEMORY_BACKEND {
        string backend_type
        bool short_term
        bool long_term
        bool entity
    }
    MODEL_CONFIG {
        string provider
        string model_key
    }
    MCP_SERVER {
        string server_url
        string server_type
    }
    EXECUTION_RUN {
        string status
        json inputs
        json result
    }
    TASK_EVENT {
        string status
        string agent_name
    }
    EXECUTION_LOG {
        text content
        datetime timestamp
    }
    EXECUTION_TRACE {
        string event_type
        json output
    }
    FLOW_EXECUTION {
        string status
        json result
    }

    USER }o--o{ GROUP : "belongs to"
    GROUP ||--o{ AGENT : "owns"
    GROUP ||--o{ TASK : "owns"
    GROUP ||--o{ CREW : "owns"
    GROUP ||--o{ TOOL : "owns"
    GROUP ||--o{ SCHEDULE : "owns"
    GROUP ||--o{ MEMORY_BACKEND : "owns"
    GROUP ||--o{ MODEL_CONFIG : "owns"
    GROUP ||--o{ MCP_SERVER : "owns"
    GROUP ||--o{ FLOW : "owns"

    AGENT ||--o{ TASK : "is assigned to"
    CREW }o--o{ AGENT : "composed of"
    CREW }o--o{ TASK : "executes"
    FLOW }o--o{ CREW : "orchestrates"

    SCHEDULE }o--|| CREW : "triggers"

    CREW ||--o{ EXECUTION_RUN : "produces"
    EXECUTION_RUN ||--o{ TASK_EVENT : "tracks"
    EXECUTION_RUN ||--o{ EXECUTION_LOG : "streams"
    EXECUTION_RUN ||--o{ EXECUTION_TRACE : "records"

    FLOW ||--o{ FLOW_EXECUTION : "produces"
```

---

## 2. Logical Data Model

Adds key attributes, data types, and relationship cardinalities. Foreign keys are shown as logical references, not necessarily enforced constraints.

```mermaid
erDiagram
    USER {
        string id PK
        string username
        string email
        enum role "admin | technical | regular"
        enum status "active | inactive | suspended"
        bool is_system_admin
        bool is_personal_workspace_manager
        datetime created_at
        datetime last_login
    }
    REFRESH_TOKEN {
        string id PK
        string user_id FK
        string token
        datetime expires_at
        bool is_revoked
    }
    GROUP {
        string id PK
        string name
        enum status "active | suspended | archived"
        string description
        bool auto_created
        string created_by_email
        datetime created_at
    }
    GROUP_USER {
        string id PK
        string group_id FK
        string user_id FK
        enum role "admin | editor | operator"
        enum status "active | inactive | suspended"
        datetime joined_at
    }
    AGENT {
        string id PK
        string name
        string role
        string goal
        string backstory
        string group_id FK
        string llm
        int temperature
        json tools
        json tool_configs
        string function_calling_llm
        int max_iter
        bool memory
        json embedder_config
        bool allow_code_execution
        string code_execution_mode
        json knowledge_sources
        datetime created_at
        datetime updated_at
    }
    TASK {
        string id PK
        string name
        string description
        string agent_id FK
        string expected_output
        string group_id FK
        json tools
        json tool_configs
        bool async_execution
        json context
        json config
        string output_json
        string output_pydantic
        string output_file
        bool markdown
        string callback
        json callback_config
        bool human_input
        string guardrail
        datetime created_at
        datetime updated_at
    }
    CREW {
        uuid id PK
        string name
        json agent_ids
        json task_ids
        json nodes
        json edges
        string group_id FK
        string created_by_email
        datetime created_at
        datetime updated_at
    }
    FLOW {
        uuid id PK
        string name
        uuid crew_id FK
        json nodes
        json edges
        json flow_config
        string group_id FK
        datetime created_at
        datetime updated_at
    }
    TOOL {
        int id PK
        string title
        string description
        string icon
        json config
        bool enabled
        string group_id FK
        datetime created_at
        datetime updated_at
    }
    SCHEDULE {
        int id PK
        string name
        string cron_expression
        json agents_yaml
        json tasks_yaml
        json inputs
        bool is_active
        bool planning
        string model
        datetime last_run_at
        datetime next_run_at
        string group_id FK
        datetime created_at
        datetime updated_at
    }
    MEMORY_BACKEND {
        string id PK
        string group_id FK
        string name
        string description
        enum backend_type "default | databricks"
        json databricks_config
        bool enable_short_term
        bool enable_long_term
        bool enable_entity
        bool enable_relationship_retrieval
        json custom_config
        bool is_active
        bool is_default
        datetime created_at
        datetime updated_at
    }
    MODEL_CONFIG {
        int id PK
        string key
        string name
        string provider
        float temperature
        int context_window
        int max_output_tokens
        bool extended_thinking
        bool enabled
        string group_id FK
        datetime created_at
        datetime updated_at
    }
    MCP_SERVER {
        int id PK
        string name
        string server_url
        string encrypted_api_key
        enum server_type "sse | streamable"
        enum auth_type "api_key | databricks_obo"
        bool enabled
        bool global_enabled
        string group_id FK
        int timeout_seconds
        int max_retries
        int rate_limit
        json additional_config
        datetime created_at
        datetime updated_at
    }
    EXECUTION_HISTORY {
        int id PK
        string job_id UK
        enum status "pending | running | completed | failed | stopped"
        json inputs
        json result
        string error
        bool planning
        string trigger_type
        datetime created_at
        datetime completed_at
        datetime stopped_at
        string stop_reason
        string stop_requested_by
        json partial_results
        bool is_stopping
        string mlflow_trace_id
        string group_id FK
        string group_email
    }
    TASK_STATUS {
        int id PK
        string job_id FK
        string task_id
        enum status "running | completed | failed"
        string agent_name
        datetime started_at
        datetime completed_at
    }
    ERROR_TRACE {
        int id PK
        int run_id FK
        string task_key
        string error_type
        string error_message
        datetime timestamp
        json error_metadata
    }
    EXECUTION_TRACE {
        int id PK
        int run_id FK
        string job_id FK
        string event_source
        string event_context
        string event_type
        json output
        json trace_metadata
        string group_id FK
        datetime created_at
    }
    EXECUTION_LOG {
        int id PK
        string execution_id
        text content
        datetime timestamp
        string group_id FK
        string group_email
    }
    FLOW_EXECUTION {
        int id PK
        uuid flow_id FK
        string job_id UK
        enum status "pending | running | completed | failed"
        json config
        json result
        text error
        datetime created_at
        datetime completed_at
    }
    FLOW_NODE_EXECUTION {
        int id PK
        int flow_execution_id FK
        string node_id
        enum status "pending | running | completed | failed"
        json result
        text error
        datetime created_at
        datetime completed_at
    }

    USER ||--o{ REFRESH_TOKEN : "has"
    USER }o--o{ GROUP : "member of via GROUP_USER"
    GROUP ||--o{ GROUP_USER : "has members"
    USER ||--o{ GROUP_USER : "assigned in"

    GROUP ||--o{ AGENT : "scopes"
    GROUP ||--o{ TASK : "scopes"
    GROUP ||--o{ CREW : "scopes"
    GROUP ||--o{ FLOW : "scopes"
    GROUP ||--o{ TOOL : "scopes"
    GROUP ||--o{ SCHEDULE : "scopes"
    GROUP ||--o{ MEMORY_BACKEND : "scopes"
    GROUP ||--o{ MODEL_CONFIG : "scopes"
    GROUP ||--o{ MCP_SERVER : "scopes"

    AGENT ||--o{ TASK : "assigned to"
    CREW ||--o{ FLOW : "referenced by"
    FLOW ||--o{ FLOW_EXECUTION : "runs as"
    FLOW_EXECUTION ||--o{ FLOW_NODE_EXECUTION : "contains"

    EXECUTION_HISTORY ||--o{ TASK_STATUS : "tracks"
    EXECUTION_HISTORY ||--o{ ERROR_TRACE : "logs errors via"
    EXECUTION_HISTORY ||--o{ EXECUTION_TRACE : "emits events to"
    EXECUTION_LOG }o--|| EXECUTION_HISTORY : "streams logs for"
```

---

## 3. Physical Data Model

Exact table names, column names, SQL types, primary keys (PK), unique keys (UK), foreign keys (FK), and notable indexes as defined in the SQLAlchemy models.

```mermaid
erDiagram
    users {
        VARCHAR id PK
        VARCHAR username UK
        VARCHAR email UK
        VARCHAR role
        VARCHAR status
        BOOLEAN is_system_admin
        BOOLEAN is_personal_workspace_manager
        TIMESTAMPTZ created_at
        TIMESTAMPTZ updated_at
        TIMESTAMPTZ last_login
    }
    refresh_tokens {
        VARCHAR id PK
        VARCHAR user_id FK
        VARCHAR token UK
        TIMESTAMPTZ expires_at
        BOOLEAN is_revoked
        TIMESTAMPTZ created_at
    }
    groups {
        VARCHAR(100) id PK
        VARCHAR(255) name
        VARCHAR(50) status
        VARCHAR(500) description
        BOOLEAN auto_created
        VARCHAR(255) created_by_email
        TIMESTAMPTZ created_at
        TIMESTAMPTZ updated_at
    }
    group_users {
        VARCHAR(100) id PK
        VARCHAR(100) group_id FK
        VARCHAR(255) user_id FK
        VARCHAR(50) role
        VARCHAR(50) status
        TIMESTAMPTZ joined_at
        BOOLEAN auto_created
        TIMESTAMPTZ created_at
        TIMESTAMPTZ updated_at
    }
    agents {
        VARCHAR id PK
        VARCHAR name
        VARCHAR role
        VARCHAR goal
        VARCHAR backstory
        VARCHAR(100) group_id IDX
        VARCHAR(255) created_by_email
        VARCHAR llm
        INTEGER temperature
        JSON tools
        JSON tool_configs
        VARCHAR function_calling_llm
        INTEGER max_iter
        INTEGER max_rpm
        INTEGER max_execution_time
        BOOLEAN verbose
        BOOLEAN allow_delegation
        BOOLEAN cache
        BOOLEAN memory
        JSON embedder_config
        VARCHAR system_template
        VARCHAR prompt_template
        VARCHAR response_template
        BOOLEAN allow_code_execution
        VARCHAR code_execution_mode
        INTEGER max_retry_limit
        BOOLEAN use_system_prompt
        BOOLEAN respect_context_window
        JSON knowledge_sources
        DATETIME created_at
        DATETIME updated_at
    }
    tasks {
        VARCHAR id PK
        VARCHAR name
        VARCHAR description
        VARCHAR agent_id FK
        VARCHAR expected_output
        JSON tools
        JSON tool_configs
        BOOLEAN async_execution
        JSON context
        JSON config
        VARCHAR(100) group_id IDX
        VARCHAR(255) created_by_email
        VARCHAR output_json
        VARCHAR output_pydantic
        VARCHAR output_file
        JSON output
        BOOLEAN markdown
        VARCHAR callback
        JSON callback_config
        BOOLEAN human_input
        VARCHAR converter_cls
        VARCHAR guardrail
        DATETIME created_at
        DATETIME updated_at
    }
    crews {
        UUID id PK
        VARCHAR name IDX
        JSON agent_ids
        JSON task_ids
        JSON nodes
        JSON edges
        VARCHAR(100) group_id IDX
        VARCHAR(255) created_by_email
        DATETIME created_at
        DATETIME updated_at
    }
    flows {
        UUID id PK
        VARCHAR name
        UUID crew_id FK
        JSON nodes
        JSON edges
        JSON flow_config
        VARCHAR(100) group_id IDX
        VARCHAR(255) created_by_email
        DATETIME created_at
        DATETIME updated_at
    }
    tools {
        INTEGER id PK
        VARCHAR title
        VARCHAR description
        VARCHAR icon
        JSON config
        BOOLEAN enabled
        VARCHAR(100) group_id IDX
        VARCHAR(255) created_by_email
        DATETIME created_at
        DATETIME updated_at
    }
    schedules {
        INTEGER id PK
        VARCHAR name
        VARCHAR cron_expression
        JSON agents_yaml
        JSON tasks_yaml
        JSON inputs
        BOOLEAN is_active
        BOOLEAN planning
        VARCHAR model
        DATETIME last_run_at
        DATETIME next_run_at
        VARCHAR(100) group_id IDX
        VARCHAR(255) created_by_email
        DATETIME created_at
        DATETIME updated_at
    }
    memory_backends {
        VARCHAR id PK
        VARCHAR(100) group_id IDX
        VARCHAR(255) name
        VARCHAR(1000) description
        VARCHAR backend_type
        JSON databricks_config
        BOOLEAN enable_short_term
        BOOLEAN enable_long_term
        BOOLEAN enable_entity
        BOOLEAN enable_relationship_retrieval
        JSON custom_config
        BOOLEAN is_active
        BOOLEAN is_default
        DATETIME created_at
        DATETIME updated_at
    }
    model_configs {
        INTEGER id PK
        VARCHAR key
        VARCHAR name
        VARCHAR provider
        FLOAT temperature
        INTEGER context_window
        INTEGER max_output_tokens
        BOOLEAN extended_thinking
        BOOLEAN enabled
        VARCHAR(100) group_id IDX
        VARCHAR(255) created_by_email
        DATETIME created_at
        DATETIME updated_at
    }
    mcp_servers {
        INTEGER id PK
        VARCHAR name
        VARCHAR server_url
        VARCHAR encrypted_api_key
        VARCHAR server_type
        VARCHAR auth_type
        BOOLEAN enabled
        BOOLEAN global_enabled
        VARCHAR group_id IDX
        INTEGER timeout_seconds
        INTEGER max_retries
        BOOLEAN model_mapping_enabled
        INTEGER rate_limit
        JSON additional_config
        DATETIME created_at
        DATETIME updated_at
    }
    executionhistory {
        INTEGER id PK
        VARCHAR job_id UK IDX
        VARCHAR status
        JSON inputs
        JSON result
        VARCHAR error
        BOOLEAN planning
        VARCHAR trigger_type
        DATETIME created_at
        DATETIME completed_at
        VARCHAR run_name
        DATETIME stopped_at
        VARCHAR stop_reason
        VARCHAR(255) stop_requested_by
        JSON partial_results
        BOOLEAN is_stopping
        VARCHAR mlflow_trace_id IDX
        VARCHAR mlflow_experiment_name
        VARCHAR mlflow_evaluation_run_id IDX
        VARCHAR(100) group_id IDX
        VARCHAR(255) group_email IDX
    }
    taskstatus {
        INTEGER id PK
        VARCHAR job_id FK IDX
        VARCHAR task_id IDX
        VARCHAR status
        VARCHAR agent_name
        DATETIME started_at
        DATETIME completed_at
    }
    errortrace {
        INTEGER id PK
        INTEGER run_id FK IDX
        VARCHAR task_key IDX
        VARCHAR error_type
        VARCHAR error_message
        TIMESTAMPTZ timestamp
        JSON error_metadata
    }
    execution_trace {
        INTEGER id PK
        INTEGER run_id FK
        VARCHAR job_id FK IDX
        VARCHAR event_source
        VARCHAR event_context
        VARCHAR event_type IDX
        JSON output
        JSON trace_metadata
        VARCHAR(100) group_id IDX
        VARCHAR(255) group_email IDX
        DATETIME created_at
    }
    execution_logs {
        INTEGER id PK
        VARCHAR execution_id IDX
        TEXT content
        DATETIME timestamp
        VARCHAR(100) group_id IDX
        VARCHAR(255) group_email IDX
    }
    flow_executions {
        INTEGER id PK
        UUID flow_id FK
        VARCHAR job_id UK
        VARCHAR status
        JSON config
        JSON result
        TEXT error
        DATETIME created_at
        DATETIME updated_at
        DATETIME completed_at
    }
    flow_node_executions {
        INTEGER id PK
        INTEGER flow_execution_id FK
        VARCHAR node_id
        VARCHAR status
        INTEGER agent_id
        INTEGER task_id
        JSON result
        TEXT error
        DATETIME created_at
        DATETIME updated_at
        DATETIME completed_at
    }

    users ||--o{ refresh_tokens : "user_id → id"
    users ||--o{ group_users : "user_id → id"
    groups ||--o{ group_users : "group_id → id"

    tasks }o--o| agents : "agent_id → id"
    flows }o--o| crews : "crew_id → id"

    executionhistory ||--o{ taskstatus : "job_id → job_id"
    executionhistory ||--o{ errortrace : "id → run_id"
    executionhistory ||--o{ execution_trace : "id → run_id"
    executionhistory ||--o{ execution_trace : "job_id → job_id"

    flow_executions }o--|| flows : "flow_id → id"
    flow_executions ||--o{ flow_node_executions : "id → flow_execution_id"
```

---

## Domain Groupings

The physical tables fall into five logical domains:

### Identity & Access
| Table | Purpose |
|-------|---------|
| `users` | Platform user accounts |
| `refresh_tokens` | JWT refresh token store |
| `groups` | Workspace/tenant groupings |
| `group_users` | User ↔ Group membership with RBAC role |

### Workflow Definition
| Table | Purpose |
|-------|---------|
| `agents` | AI agent definitions (role, goal, LLM, tools) |
| `tasks` | Task definitions (description, expected output, tools) |
| `crews` | Agent+task compositions with visual topology (nodes/edges) |
| `flows` | Multi-crew orchestration graphs |

### Execution & Observability
| Table | Purpose |
|-------|---------|
| `executionhistory` | Lifecycle record for each crew run |
| `taskstatus` | Per-task status within a run |
| `errortrace` | Structured error capture per run |
| `execution_trace` | Granular event trace (agent thoughts, tool calls, outputs) |
| `execution_logs` | Raw text log stream per execution |
| `flow_executions` | Flow-level execution record |
| `flow_node_executions` | Per-node execution tracking within a flow |

### Configuration
| Table | Purpose |
|-------|---------|
| `tools` | Custom tool registry |
| `schedules` | Cron-based execution schedules |
| `memory_backends` | Vector memory backend configuration |
| `model_configs` | LLM model definitions (provider, limits, settings) |
| `mcp_servers` | MCP tool server connections |

### Key Design Notes

- **Group isolation**: Every domain entity has a `group_id` column. All queries are scoped to the current group extracted from the request context.
- **JSON arrays for crew composition**: `crews.agent_ids` and `crews.task_ids` are JSON arrays, not foreign-key join tables. This enables flexible ordering without a junction table.
- **Dual foreign keys in execution_trace**: References `executionhistory` by both `id` (integer PK) and `job_id` (string UUID) to support different lookup patterns.
- **No hardcoded cascade deletes** on most entities — deletion is handled at the service layer to allow soft deletes and audit logging.
- **UUIDs vs integers**: Workflow entities (agents, tasks, crews, flows) use UUID primary keys. Operational/log entities (executionhistory, taskstatus, errortrace, etc.) use auto-increment integers for insert performance.
