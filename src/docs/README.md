
# Kasal Documentation Hub

**Enterprise AI Agent Workflow Orchestration Platform**

---

## What Is Kasal?

Kasal is an enterprise-grade platform for designing, deploying, and monitoring autonomous AI agent workflows. It provides a drag-and-drop visual designer where users compose teams of AI agents, define tasks, and orchestrate multi-step pipelines — all backed by the [CrewAI](https://crewai.com) engine and deeply integrated with Databricks.

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/databrickslabs/kasal

# Start the backend (uv syncs dependencies automatically)
cd kasal/src/backend && ./run.sh

# In another terminal, start the frontend
cd kasal/src/frontend && npm install && npm start
```

Access the application at `http://localhost:3000`

---

## Documentation Index

### Architecture & Design
| Document | Audience | Description |
|----------|----------|-------------|
| [Architecture Guide](./ARCHITECTURE_GUIDE.md) | Architects, Tech Leads | System design, layers, patterns, security model |
| [Code Structure Guide](./CODE_STRUCTURE_GUIDE.md) | Developers | File layout, module responsibilities, naming conventions |
| [Data Models](./DATA_MODELS.md) | Architects, DBAs, Developers | Conceptual, logical, and physical data model diagrams |
| [Data Architecture Template](./DATA_ARCHITECTURE_TEMPLATE.md) | Architects, DBAs | Reusable template for data architecture documentation |
| [Documentation Structure](./DOCUMENTATION_STRUCTURE.md) | All | Conceptual, logical, and physical diagrams of the docs folder itself |

### Development
| Document | Audience | Description |
|----------|----------|-------------|
| [Developer Guide](./DEVELOPER_GUIDE.md) | Backend & Frontend Engineers | Setup, workflows, coding conventions, testing |
| [Configuration Guide](./CONFIGURATION_GUIDE.md) | DevOps, Developers | Environment variables, database, auth, LLM settings |

### API & Security
| Document | Audience | Description |
|----------|----------|-------------|
| [API Reference](./API_REFERENCE.md) | API Integrators | REST endpoints, WebSocket events, request/response examples |
| [Security Guide](./SECURITY_GUIDE.md) | Security Engineers, Admins | Auth, RBAC, multi-tenancy, secrets management |

### User Guides
| Document | Audience | Description |
|----------|----------|-------------|
| [End User Tutorial Catalog](./END_USER_TUTORIAL_CATALOG.md) | End Users | Step-by-step tutorials for building workflows |
| [Why Kasal](./WHY_KASAL.md) | Stakeholders | Problem statement, value proposition |

### Legacy / Archive
Older deep-dive documents are in [archive/](./archive/) and may be outdated but contain useful historical context.

---

## Key Features

| Feature | Description |
|---------|-------------|
| **Visual Workflow Designer** | Drag-and-drop ReactFlow canvas to compose agent crews and task graphs |
| **AI Agent Orchestration** | CrewAI-powered sequential and hierarchical crew execution |
| **Multi-Model Support** | GPT-4, Claude, Databricks Foundation Models, Llama, and more via LiteLLM |
| **Databricks Integration** | Native connectors for Genie, Jobs, Vector Search, Secrets, Volumes |
| **Memory & Knowledge** | Configurable short-term, long-term, and entity memory with Vector Search |
| **MCP Tool Support** | Model Context Protocol integration for extensible custom tooling |
| **Real-time Monitoring** | Live execution logs, task traces, and status tracking |
| **Scheduling** | Cron-based job scheduling for recurring workflows |
| **Multi-Tenancy** | Group-based workspace isolation with role-based access control |
| **DSPy Optimization** | Optional prompt optimization via DSPy framework |

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| **Backend** | FastAPI 0.110+, SQLAlchemy 2.0+, Alembic, Python 3.10+ |
| **Frontend** | React 18, TypeScript 4.9, Material-UI 5.17, ReactFlow 11.10 |
| **AI Engine** | CrewAI 0.193+, LiteLLM 1.74+, LangChain |
| **Database** | PostgreSQL (production) / SQLite (development) |
| **Authentication** | JWT + Databricks OAuth 2.0 |
| **Observability** | MLflow 3.4, structured logging |
| **Package Manager** | uv (backend), npm (frontend) |

---

## Project Structure

```
kasal/
├── src/
│   ├── backend/              # FastAPI Python backend
│   │   ├── src/              # Application source code
│   │   ├── tests/            # Unit and integration tests
│   │   ├── migrations/       # Alembic database migrations
│   │   └── pyproject.toml    # Python dependencies
│   ├── frontend/             # React TypeScript frontend
│   │   ├── src/              # React application source
│   │   └── package.json      # npm dependencies
│   ├── frontend_static/      # Built frontend assets
│   └── docs/                 # This documentation
├── README.md                 # Project overview
├── CONTRIBUTING.md           # Contribution guidelines
└── manifest.yaml             # Databricks Apps manifest
```

---

## Version Information

- **Current Version**: 2.0.0
- **Documentation Updated**: April 2026
- **Minimum Python Version**: 3.10
- **Minimum Node Version**: 18.0

---

## Support and Resources

- **GitHub Issues**: [github.com/databrickslabs/kasal/issues](https://github.com/databrickslabs/kasal/issues)
- **Contributing**: [CONTRIBUTING.md](https://github.com/databrickslabs/kasal/blob/main/CONTRIBUTING.md)
- **License**: Apache License 2.0

---

*Built by Databricks Labs*
