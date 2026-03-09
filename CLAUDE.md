# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Truss is a Django 5.2 web application (Python 3.13) scaffolded with Cookiecutter Django. It uses Docker Compose for local development with PostgreSQL, Redis, Celery, and a Node/Webpack frontend pipeline.

## Development Environment

All services run in Docker. The `justfile` is the primary task runner (not Make):

```bash
just build          # Build the Django Docker image
just up             # Start all containers (detached)
just down           # Stop containers
just logs           # Tail container logs (accepts service name arg)
just manage <cmd>   # Run manage.py inside the Django container
just prune          # Remove containers AND volumes (destructive)
```

The compose file used is `docker-compose.local.yml` (set via `COMPOSE_FILE` in the justfile).

### Running Tests

```bash
# Full test suite (inside Docker)
docker compose -f docker-compose.local.yml run django pytest

# Single test file
docker compose -f docker-compose.local.yml run django pytest truss/users/tests/test_models.py

# Single test
docker compose -f docker-compose.local.yml run django pytest truss/users/tests/test_models.py::test_user_get_absolute_url

# With uv locally (requires local DB/Redis)
uv run pytest
uv run pytest truss/users/tests/test_views.py -k "test_detail"
```

Pytest is configured in `pyproject.toml` with `--ds=config.settings.test --reuse-db --import-mode=importlib`.

### Linting and Type Checking

```bash
uv run ruff check .              # Lint
uv run ruff check --fix .        # Lint with auto-fix
uv run ruff format .             # Format
uv run mypy truss                # Type check
uv run djlint truss/templates/   # Lint Django templates
pre-commit run --all-files       # Run all pre-commit hooks
```

Pre-commit hooks: trailing-whitespace, end-of-file-fixer, prettier (HTML/JS/CSS), django-upgrade (target 5.2), ruff (check + format), pyproject-fmt, djlint.

### Coverage

```bash
uv run coverage run -m pytest
uv run coverage html
```

## Architecture

### Django Project Layout

- `config/` — Project configuration (settings, root URLs, ASGI/WSGI, Celery app)
  - `config/settings/base.py` — Shared settings
  - `config/settings/local.py` — Development overrides (debug toolbar, extensions)
  - `config/settings/test.py` — Test overrides (MD5 hasher, fake webpack loader)
  - `config/settings/production.py` — Production settings
  - `config/api_router.py` — DRF router; register new API viewsets here
  - `config/urls.py` — Root URL configuration
- `truss/` — Application code (`APPS_DIR`)
  - `truss/users/` — Custom user app (email-based auth, no username)
  - `truss/users/api/` — DRF serializers and viewsets for users
  - `truss/contrib/sites/` — Custom site migrations
  - `truss/templates/` — Django templates (allauth overrides)
  - `truss/static/` — Static assets (Sass, fonts, favicons)
  - `truss/conftest.py` — Shared pytest fixtures (`user` factory)

### Key Design Decisions

- **Custom User model** (`truss.users.models.User`): Uses email as `USERNAME_FIELD`, no username. Single `name` field instead of first/last. Managed by custom `UserManager`.
- **Authentication**: django-allauth with email-only login (`ACCOUNT_LOGIN_METHODS = {"email"}`), mandatory email verification, MFA support.
- **API**: Django REST Framework with token + session auth, drf-spectacular for OpenAPI schema. API docs at `/api/docs/`, schema at `/api/schema/`.
- **Task queue**: Celery with Redis broker, django-celery-beat for periodic tasks.
- **Frontend**: Webpack (dev server on port 3000), Bootstrap 5 with custom Sass, django-webpack-loader integration.
- **ASGI**: Uvicorn with basic WebSocket support (`config/websocket.py`).

### Docker Services (local)

| Service | Port | Purpose |
|---------|------|---------|
| django | 8000 | Web server |
| postgres | — | PostgreSQL database |
| redis | — | Cache + Celery broker |
| mailpit | 8025 | Email testing UI |
| celeryworker | — | Async task processing |
| celerybeat | — | Periodic task scheduler |
| flower | 5555 | Celery monitoring |
| node | 3000 | Webpack dev server |

### Adding New Apps

1. Create the app under `truss/` (e.g., `truss/myapp/`)
2. Add to `LOCAL_APPS` in `config/settings/base.py`
3. Register API viewsets in `config/api_router.py`
4. Add URL patterns in `config/urls.py`

### Environment Variables

Stored in `.envs/.local/.django` and `.envs/.local/.postgres` (not committed). Key vars: `DATABASE_URL`, `REDIS_URL`, `DJANGO_SETTINGS_MODULE`, `USE_DOCKER`.

## Conventions

- **Package manager**: `uv` (not pip)
- **Linter/formatter**: Ruff (see `pyproject.toml [tool.ruff]` for enabled rules)
- **Template linter**: djLint (profile: django)
- **Tests**: pytest + factory_boy (factories in each app's `tests/factories.py`)
- **Imports**: Ruff isort with `force-single-line = true` — one import per line
- **Django target**: 5.2 (enforced by django-upgrade pre-commit hook)
