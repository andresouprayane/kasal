# Kasal Configuration Guide

**Audience**: Developers, DevOps, System Administrators

---

## Overview

Kasal is configured entirely through environment variables. There are no configuration files to edit — all settings flow through `src/backend/src/config/settings.py`, which uses Pydantic Settings to parse and validate the environment.

---

## Environment Variable Reference

### Database

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_TYPE` | `postgres` | Database backend: `postgres` or `sqlite` |
| `SQLITE_DB_PATH` | `./app.db` | Path to SQLite file (used when `DATABASE_TYPE=sqlite`) |
| `POSTGRES_HOST` | `localhost` | PostgreSQL host |
| `POSTGRES_PORT` | `5432` | PostgreSQL port |
| `POSTGRES_DB` | `kasal` | PostgreSQL database name |
| `POSTGRES_USER` | `kasal` | PostgreSQL username |
| `POSTGRES_PASSWORD` | *(required)* | PostgreSQL password |
| `DATABASE_URI` | *(auto-assembled)* | Full connection string; overrides individual Postgres settings |
| `USE_NULLPOOL` | `false` | Disable connection pooling (useful for serverless/async contexts) |

**Development** — use SQLite:
```bash
DATABASE_TYPE=sqlite
SQLITE_DB_PATH=./app.db
```

**Production** — use PostgreSQL:
```bash
DATABASE_TYPE=postgres
POSTGRES_HOST=your-db-host
POSTGRES_PORT=5432
POSTGRES_DB=kasal
POSTGRES_USER=kasal
POSTGRES_PASSWORD=your-secure-password
```

---

### Authentication

| Variable | Default | Description |
|----------|---------|-------------|
| `SECRET_KEY` | *(required)* | JWT signing key — use a long random string in production |
| `ALGORITHM` | `HS256` | JWT signing algorithm |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `11520` | Access token lifetime (default: 8 days) |
| `REFRESH_TOKEN_EXPIRE_DAYS` | `30` | Refresh token lifetime |

Generate a secure secret key:
```bash
python -c "import secrets; print(secrets.token_hex(64))"
```

---

### Application Behavior

| Variable | Default | Description |
|----------|---------|-------------|
| `LOG_LEVEL` | `INFO` | Logging verbosity: `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `AUTO_SEED_DATABASE` | `true` | Run database seeders at startup (model configs, tools) |
| `DOCS_ENABLED` | `true` | Expose OpenAPI docs at `/api-docs` |
| `CORS_ORIGINS` | `["*"]` | Allowed CORS origins (comma-separated or JSON list) |
| `API_PREFIX` | `/api/v1` | URL prefix for all API routes |

---

### Databricks Integration

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABRICKS_HOST` | *(optional)* | Databricks workspace URL (e.g., `https://adb-xxx.azuredatabricks.net`) |
| `DATABRICKS_TOKEN` | *(optional)* | Personal access token for Databricks API calls |
| `DATABRICKS_HTTP_PATH` | *(optional)* | SQL warehouse HTTP path for Lakebase integration |
| `DATABRICKS_WAREHOUSE_ID` | *(optional)* | SQL warehouse ID |
| `DATABRICKS_CATALOG` | *(optional)* | Unity Catalog catalog name |
| `DATABRICKS_SCHEMA` | *(optional)* | Unity Catalog schema name |

In Databricks Apps deployments, credentials are injected via service principal — no need to set these manually.

---

### LLM / Model Configuration

LLM credentials are typically managed per-model-config in the database (via the Configuration UI or seeds), not globally. However, LiteLLM respects standard provider environment variables:

| Variable | Provider |
|----------|---------|
| `OPENAI_API_KEY` | OpenAI |
| `ANTHROPIC_API_KEY` | Anthropic |
| `AZURE_API_KEY`, `AZURE_API_BASE` | Azure OpenAI |

For Databricks models, the `DATABRICKS_HOST` and `DATABRICKS_TOKEN` above are used.

---

### MLflow

| Variable | Default | Description |
|----------|---------|-------------|
| `MLFLOW_TRACKING_URI` | *(optional)* | MLflow tracking server URL |
| `MLFLOW_EXPERIMENT_NAME` | `kasal` | Default experiment name |

---

### Memory / Vector Search

| Variable | Default | Description |
|----------|---------|-------------|
| `VECTOR_SEARCH_ENDPOINT` | *(optional)* | Databricks Vector Search endpoint name |
| `EMBEDDINGS_MODEL` | *(optional)* | Default embeddings model for memory |

---

## Development Setup

Create a `.env` file in `src/backend/` for local development:

```bash
# Database
DATABASE_TYPE=sqlite
SQLITE_DB_PATH=./app.db

# Auth
SECRET_KEY=dev-secret-key-change-in-production

# Logging
LOG_LEVEL=DEBUG

# Seed on startup
AUTO_SEED_DATABASE=true
```

The backend loads `.env` automatically via Pydantic Settings.

---

## Production Setup (Databricks Apps)

For Databricks Apps deployment, configuration is managed through `manifest.yaml` and Databricks Secrets:

1. **Secrets**: Store sensitive values in Databricks Secret Scopes
2. **manifest.yaml**: Reference secrets as environment variable bindings
3. **Service Principal**: Databricks injects identity headers; no token needed

Example `manifest.yaml` environment binding:
```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretScope: kasal-secrets
      secretKey: postgres-password
  - name: SECRET_KEY
    valueFrom:
      secretScope: kasal-secrets
      secretKey: jwt-secret
```

---

## Frontend Configuration

The frontend is configured at build time via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `REACT_APP_API_URL` | `http://localhost:8000/api/v1` | Backend API base URL |
| `REACT_APP_WS_URL` | `ws://localhost:8000` | WebSocket base URL |

For production builds:
```bash
REACT_APP_API_URL=https://your-kasal-backend/api/v1 npm run build
```

Or use `src/build.py` which handles the full build and copies assets to `frontend_static/`.

---

## Database Initialization

On first startup with `AUTO_SEED_DATABASE=true`, Kasal:
1. Runs all pending Alembic migrations to create schema
2. Seeds default model configurations (Databricks, OpenAI, Anthropic models)
3. Seeds default tool definitions
4. Creates a default admin user if none exists

To run migrations manually:
```bash
cd src/backend
uv run alembic upgrade head
```

---

## CORS Configuration

For development, CORS allows all origins. For production, restrict to your frontend domain:

```bash
CORS_ORIGINS=["https://your-app.databricks.com","https://localhost:3000"]
```

---

## Logging Configuration

Kasal uses Python's `logging` module with structured JSON output. Set the level via `LOG_LEVEL`:

```bash
# Verbose debugging (shows SQL queries, HTTP calls)
LOG_LEVEL=DEBUG

# Production default
LOG_LEVEL=INFO

# Errors only
LOG_LEVEL=ERROR
```

Log output goes to stdout and is captured by Databricks Apps logging infrastructure in production.
