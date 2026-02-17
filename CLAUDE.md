# CLAUDE.md - Open WebUI Developer Guide

## Project Overview

Open WebUI is a self-hosted AI chat interface (v0.8.3). It has a **SvelteKit frontend** (TypeScript/Svelte 5) and a **FastAPI backend** (Python 3.11+). The frontend is a client-side rendered SPA (SSR disabled) that communicates with the backend via REST APIs and Socket.IO for real-time features.

## Repository Structure

```
open-webui/
├── src/                    # Frontend (SvelteKit + TypeScript)
│   ├── lib/
│   │   ├── apis/           # Backend API client modules (one per resource)
│   │   ├── components/     # Svelte components (feature-first organization)
│   │   │   ├── admin/      # Admin panel (Analytics, Settings, Users)
│   │   │   ├── chat/       # Chat interface (Messages, MessageInput, Markdown)
│   │   │   ├── channel/    # Team channels
│   │   │   ├── common/     # Shared UI components
│   │   │   ├── icons/      # SVG icon components
│   │   │   ├── layout/     # Sidebar, Navbar, Modals
│   │   │   ├── notes/      # Notes feature
│   │   │   ├── playground/  # Playground/testing
│   │   │   └── workspace/  # Knowledge, Models, Prompts, Tools
│   │   ├── stores/         # Svelte writable stores (index.ts - centralized)
│   │   ├── i18n/           # i18next with 60+ locales
│   │   ├── types/          # TypeScript type definitions
│   │   ├── utils/          # Utility functions
│   │   ├── workers/        # Web Workers (Pyodide, TTS)
│   │   └── constants.ts    # API base URLs, app constants
│   └── routes/             # SvelteKit file-based routing
│       ├── (app)/          # Authenticated app layout group
│       │   ├── c/[id]/     # Chat page
│       │   ├── admin/      # Admin area
│       │   └── workspace/  # Workspace management
│       └── auth/           # Authentication page
├── backend/
│   └── open_webui/         # Python package
│       ├── main.py         # FastAPI app entry point
│       ├── config.py       # Configuration management (~4100 lines)
│       ├── env.py          # Environment variable loading (~990 lines)
│       ├── constants.py    # Error messages, enums
│       ├── models/         # SQLAlchemy ORM models + Pydantic schemas
│       ├── routers/        # API route handlers (26 modules)
│       ├── utils/          # Auth, middleware, access control, etc.
│       ├── internal/       # Database setup (db.py)
│       ├── retrieval/      # RAG pipeline, vector DBs, document loaders
│       ├── storage/        # Storage provider abstraction (local/S3/GCS/Azure)
│       ├── socket/         # Socket.IO real-time handler
│       ├── migrations/     # Alembic database migrations
│       └── test/           # Backend pytest tests
├── cypress/                # E2E tests (Cypress)
├── static/                 # Static assets
├── scripts/                # Build scripts (prepare-pyodide.js)
└── docs/                   # Documentation
```

## Quick Commands

### Frontend Development
```bash
npm install              # Install frontend dependencies (use --force if needed)
npm run dev              # Start Vite dev server with Pyodide + hot reload
npm run build            # Production build to build/
npm run check            # SvelteKit type checking (svelte-check)
npm run lint:frontend    # ESLint with auto-fix
npm run format           # Prettier on all frontend files
npm run test:frontend    # Vitest unit tests
npm run cy:open          # Cypress E2E interactive runner
```

### Backend Development
```bash
cd backend && bash dev.sh                   # Start backend with uvicorn --reload on port 8080
pip install -e ".[all]"                     # Install backend in editable mode with all extras
npm run lint:backend                        # Pylint on backend/
npm run format:backend                      # Black formatter on Python code
pytest backend/                             # Run backend tests
```

### Combined
```bash
npm run lint             # Run all linting (frontend + types + backend)
```

### Docker
```bash
docker compose up -d                        # Default deployment
docker compose -f docker-compose.yaml -f docker-compose.gpu.yaml up -d  # With NVIDIA GPU
```

## Architecture & Key Patterns

### Frontend

- **Framework**: SvelteKit 2.5 + Svelte 5 + TypeScript 5.5 (strict mode)
- **Styling**: Tailwind CSS 4.0 with class-based dark mode
- **State**: Centralized Svelte writable stores in `src/lib/stores/index.ts` (~50+ stores)
- **API layer**: Each resource has a module in `src/lib/apis/` using native `fetch()` with Bearer token auth
- **Routing**: SvelteKit file-based routing, SSR disabled (client-side SPA)
- **i18n**: i18next with browser language detection, lazy-loaded locale JSON files
- **Rich text**: Tiptap editor + CodeMirror for code
- **Markdown**: marked library with custom extensions (KaTeX, citations, mermaid)
- **Real-time**: Socket.IO client for presence, chat events, collaborative editing (Yjs CRDT)

**Frontend API function pattern:**
```typescript
export const doSomething = async (token: string, params: object) => {
    let error = null;
    const res = await fetch(`${WEBUI_API_BASE_URL}/endpoint`, {
        method: 'POST',
        headers: {
            Accept: 'application/json',
            'Content-Type': 'application/json',
            authorization: `Bearer ${token}`
        },
        body: JSON.stringify(params)
    })
        .then(async (res) => {
            if (!res.ok) throw await res.json();
            return res.json();
        })
        .catch((err) => {
            error = err;
            return null;
        });
    if (error) throw error;
    return res;
};
```

### Backend

- **Framework**: FastAPI 0.128 with Uvicorn
- **Database**: SQLAlchemy 2.0 ORM with Alembic migrations; supports SQLite (default), PostgreSQL, MySQL
- **Auth**: JWT (HS256) with bcrypt/argon2 password hashing; OAuth/OIDC, LDAP, trusted headers supported
- **Authorization**: Role-based (admin/user/pending) + group-based permissions via `has_permission()`
- **Sessions**: HTTPOnly cookies, optionally Redis-backed
- **Config**: Environment variables loaded via python-dotenv; dynamic config cached in Redis (`AppConfig`)
- **Real-time**: python-socketio with Redis pub/sub for multi-instance
- **RAG**: LangChain + multiple vector DB backends (ChromaDB, Qdrant, Weaviate, Elasticsearch, Pinecone, pgvector)
- **Storage**: Pluggable providers - local filesystem, S3, GCS, Azure Blob

**API routes are mounted at:**
- `/api/v1/*` - Main REST API (auth, users, chats, models, files, knowledge, etc.)
- `/ollama` - Ollama API proxy
- `/openai` - OpenAI-compatible API proxy
- `/api/chat/completions` - OpenAI-compatible chat completions endpoint

**Backend model pattern** (each file in `backend/open_webui/models/`):
- SQLAlchemy `Base` model class (DB schema)
- Pydantic `BaseModel` classes (request/response schemas)
- A "table" class with static methods for CRUD operations (e.g., `Users`, `Chats`)

**Router pattern** (each file in `backend/open_webui/routers/`):
- `APIRouter()` instance
- FastAPI `Depends()` for auth (`get_verified_user`, `get_admin_user`) and DB sessions (`get_session`)
- Pydantic models for request validation

## Code Style & Conventions

### Frontend (TypeScript/Svelte)
- **Formatter**: Prettier - tabs, single quotes, no trailing commas, 100 char print width, LF line endings
- **Linter**: ESLint with @typescript-eslint, svelte plugin, cypress plugin, prettier integration
- **Components**: Feature-first directory organization; large features use subdirectories
- **Stores**: All in one file (`stores/index.ts`), use Svelte `writable()` stores
- **Imports**: Use `$lib/` alias for `src/lib/` paths

### Backend (Python)
- **Formatter**: Black (default settings, excludes .venv)
- **Linter**: Pylint
- **Types**: Pydantic v2 for all data validation and serialization
- **Async**: Most endpoints are async; use `async def` for route handlers
- **Logging**: loguru library (`from loguru import logger`)
- **DB sessions**: Use `Depends(get_session)` for request-scoped sessions; `get_db_context()` for utility code

## Environment Variables

Configuration is environment-driven. Key variables:

| Variable | Purpose | Default |
|----------|---------|---------|
| `DATABASE_URL` | Database connection string | `sqlite:///webui.db` |
| `WEBUI_SECRET_KEY` | JWT signing key | Auto-generated |
| `OLLAMA_BASE_URLS` | Ollama server URLs | `http://localhost:11434` |
| `OPENAI_API_KEYS` | OpenAI API keys | - |
| `WEBUI_AUTH` | Enable authentication | `true` |
| `ENABLE_SIGNUP` | Allow user registration | `true` |
| `PORT` | Server port | `8080` |
| `UVICORN_WORKERS` | Backend worker count | `1` |
| `ENABLE_RAG` | Enable RAG features | `true` |
| `RAG_EMBEDDING_MODEL` | Embedding model name | `sentence-transformers/all-MiniLM-L6-v2` |

See `backend/open_webui/env.py` and `backend/open_webui/config.py` for the full list (400+ variables).

## Testing

- **Frontend unit tests**: Vitest (`npm run test:frontend`)
- **Frontend E2E**: Cypress (`npm run cy:open`), tests in `cypress/e2e/`
- **Backend tests**: pytest (`pytest backend/`), test files in `backend/open_webui/test/`
- **CI checks**: GitHub Actions run format verification (no auto-fix, checks git diff), type checking, and builds on push/PR to main/dev

## Database Migrations

- Managed by **Alembic** in `backend/open_webui/migrations/`
- Migrations run automatically on startup when `ENABLE_DB_MIGRATIONS=true`
- Legacy Peewee migrations are also supported for backward compatibility
- When adding new models, create an Alembic migration: `alembic revision --autogenerate -m "description"`

## Key Files Reference

| File | Purpose |
|------|---------|
| `backend/open_webui/main.py` | FastAPI app setup, middleware stack, router registration |
| `backend/open_webui/config.py` | All configuration variables and AppConfig class |
| `backend/open_webui/env.py` | Environment variable loading and path setup |
| `backend/open_webui/internal/db.py` | Database engine, session management, migration runner |
| `backend/open_webui/utils/auth.py` | JWT creation/validation, password hashing |
| `backend/open_webui/utils/access_control.py` | Permission checking, group-based access |
| `backend/open_webui/socket/main.py` | Socket.IO event handlers, real-time features |
| `src/lib/stores/index.ts` | All frontend Svelte stores |
| `src/lib/constants.ts` | Frontend API base URLs and constants |
| `src/lib/apis/` | Frontend API client functions (one file per resource) |
| `src/routes/(app)/+layout.svelte` | Main app layout with sidebar and initialization |

## Common Development Tasks

**Adding a new API endpoint:**
1. Add/update the SQLAlchemy model in `backend/open_webui/models/`
2. Add Pydantic request/response schemas in the same file
3. Create or update the router in `backend/open_webui/routers/`
4. Register the router in `backend/open_webui/main.py` if new
5. Add the corresponding frontend API function in `src/lib/apis/`
6. Create Alembic migration if DB schema changed

**Adding a new frontend page:**
1. Create route directory under `src/routes/(app)/`
2. Add `+page.svelte` (and optionally `+page.ts` for data loading)
3. Components go in `src/lib/components/<feature>/`
4. Add any needed stores to `src/lib/stores/index.ts`

**Adding translations:**
1. Add translation keys to `src/lib/i18n/locales/en-US/translation.json`
2. Run `npm run i18n:parse` to sync all locale files
3. Use `$i18n.t('key')` in Svelte components
