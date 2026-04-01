# Kasal Security Guide

**Audience**: Security Engineers, System Administrators, Architects

---

## Authentication

### JWT-Based Authentication

Kasal uses JSON Web Tokens (JWT) for API authentication.

**Token Types**:

| Token | Lifetime | Storage | Purpose |
|-------|---------|---------|---------|
| Access Token | 8 days | Client memory / Authorization header | API request authentication |
| Refresh Token | 30 days | httpOnly cookie (secure, SameSite) | Access token renewal |

**Login Flow**:
```
POST /api/v1/auth/login
  Body: { username, password }
  Response: { access_token, token_type }
  Cookie: refresh_token (httpOnly)
```

**Using the Access Token**:
```
Authorization: Bearer <access_token>
```

**Refreshing Tokens**:
```
POST /api/v1/auth/refresh-token
  Cookie: refresh_token (automatically sent)
  Response: { access_token }
```

**Logout**:
```
POST /api/v1/auth/logout
  Clears refresh_token cookie
```

### Databricks OAuth

For Databricks Apps deployments, Kasal supports OAuth 2.0 via the Databricks identity provider:

```
POST /api/v1/auth/oauth/{provider}/authorize
  Response: { authorization_url }

POST /api/v1/auth/oauth/{provider}/callback
  Body: { code, state }
  Response: { access_token }
```

The OAuth flow creates or updates a platform user record and issues a Kasal JWT for subsequent API calls.

### Token Security

- Access tokens are signed with `SECRET_KEY` using the `HS256` algorithm
- Refresh tokens are stored as httpOnly cookies (not accessible to JavaScript)
- All token operations require HTTPS in production

---

## Role-Based Access Control (RBAC)

### Role Hierarchy

```
System Admin
    │
    ├── Workspace Admin
    │       │
    │       ├── Editor
    │       │       │
    │       │       └── Operator
    │
    └── (all roles below)
```

### Permission Matrix

| Capability | Operator | Editor | Workspace Admin | System Admin |
|-----------|---------|--------|----------------|-------------|
| View executions | ✓ | ✓ | ✓ | ✓ |
| Trigger executions | ✓ | ✓ | ✓ | ✓ |
| View logs and traces | ✓ | ✓ | ✓ | ✓ |
| Create/edit agents | | ✓ | ✓ | ✓ |
| Create/edit tasks | | ✓ | ✓ | ✓ |
| Create/edit crews | | ✓ | ✓ | ✓ |
| Manage tools | | ✓ | ✓ | ✓ |
| Configure engine/models | | ✓ | ✓ | ✓ |
| Manage schedules | | ✓ | ✓ | ✓ |
| Manage workspace users | | | ✓ | ✓ |
| Manage memory backends | | | ✓ | ✓ |
| System configuration | | | | ✓ |
| Cross-workspace access | | | | ✓ |

### Role Assignment

Roles are assigned per workspace (group). A user can have different roles in different workspaces. The effective role for a request is determined by:

1. If `system_admin = true` on the user record → System Admin globally
2. If the user is workspace admin for the current group → Workspace Admin
3. Otherwise → the assigned role for the current group

### Permission Enforcement

Permissions are enforced in `src/backend/src/core/permissions.py` via a `check_permission()` decorator applied to service methods:

```python
@check_permission(Permission.EDIT_AGENTS)
async def create_agent(self, agent_data: AgentCreate, group_id: str) -> Agent:
    ...
```

The decorator retrieves the current user's effective role and raises `PermissionDeniedError` if the role lacks the required permission.

---

## Multi-Tenancy and Data Isolation

### Group-Based Tenancy

Every user belongs to one or more workspace **groups**. All data entities (agents, tasks, crews, executions, tools, etc.) are scoped to a `group_id`.

### Isolation Enforcement

1. **Middleware** (`middleware/group_context.py`): Extracts `group_id` from request context (Databricks headers or JWT claims) and injects it into the request state
2. **Dependency** (`Depends(get_current_group)`): Routes use this to get the scoped group ID
3. **Repository Layer**: Every query includes a `WHERE group_id = :group_id` filter
4. **No cross-group access**: Users cannot read or modify data from other groups unless they are System Admin

### Databricks Apps Identity

When running as a Databricks App:
- User identity is extracted from the `X-Databricks-User` header (set by the Databricks Apps proxy)
- This header is trusted at the middleware level
- No additional authentication is required — the platform handles it

---

## Secrets Management

### Development

Use a `.env` file in `src/backend/` for local secrets. This file is gitignored.

### Production (Databricks Apps)

Store secrets in **Databricks Secret Scopes** and reference them in `manifest.yaml`:

```yaml
env:
  - name: SECRET_KEY
    valueFrom:
      secretScope: kasal-prod
      secretKey: jwt-secret-key
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretScope: kasal-prod
      secretKey: db-password
```

Never hardcode secrets in code or configuration files.

### LLM API Keys

LLM provider API keys are stored:
- **Development**: In `.env` as `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.
- **Production**: In Databricks Secret Scopes, referenced by model configurations
- **Databricks Models**: Authenticated via the service principal — no API key needed

---

## Input Validation

### Backend Validation

All API inputs are validated by Pydantic V2 schemas before reaching service code. This provides:
- Type coercion and validation
- Required field enforcement
- Custom validators for business rules

### SQL Injection Prevention

Kasal uses SQLAlchemy 2.0 with parameterized queries exclusively. Raw SQL strings are never constructed from user input.

### No External URL Exposure

The codebase does not embed real service URLs. All external endpoints are configured via environment variables at runtime.

---

## Transport Security

### HTTPS

In production (Databricks Apps), all traffic is terminated via HTTPS by the Databricks Apps proxy. The backend itself does not need to handle TLS.

For standalone deployments, place Kasal behind a reverse proxy (nginx, Caddy, AWS ALB) that handles TLS termination.

### CORS

CORS is configured via the `CORS_ORIGINS` environment variable. In production, restrict to known origins:

```bash
CORS_ORIGINS=["https://your-app.databricks.com"]
```

Development defaults to allowing all origins (`*`).

---

## Audit and Observability

### Execution Logging

All execution events are logged to `ExecutionLogs` including:
- Agent thoughts and decisions
- Tool invocations (input and output)
- Task completions
- Errors and exceptions

### Request Logging

The `logging_middleware.py` logs all API requests with:
- HTTP method and path
- Response status code
- Request duration
- User/group context

### Error Traces

`ErrorTrace` records capture structured error information including:
- Error type and message
- Task context at time of failure
- Metadata for debugging

---

## Security Checklist for Deployments

- [ ] `SECRET_KEY` is a strong random string (≥64 hex characters)
- [ ] `SECRET_KEY` is stored in Databricks Secrets, not in code
- [ ] Database password stored in Databricks Secrets
- [ ] CORS restricted to known production origins
- [ ] `DOCS_ENABLED=false` in production (disables `/api-docs`)
- [ ] All traffic served over HTTPS
- [ ] Databricks Apps proxy handles user identity (no additional auth layer needed)
- [ ] LLM API keys stored in Databricks Secrets
- [ ] Log level set to `INFO` or higher in production (not `DEBUG`)
- [ ] Database user has minimal required permissions (no superuser)
