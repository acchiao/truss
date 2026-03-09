# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Truss is a Django 5.2 web application (Python 3.13) scaffolded with Cookiecutter Django. It uses Docker Compose for local development with PostgreSQL, Redis, Celery, and a Node/Webpack frontend pipeline. Production uses Sentry for error tracking and Anymail/SendGrid for email.

## Development Environment

All services run in Docker. The `justfile` is the primary task runner (not Make):

```bash
just build              # Build the Django Docker image
just up                 # Start all containers (detached)
just down               # Stop containers
just logs               # Tail container logs (accepts service name arg)
just manage <cmd>       # Run manage.py inside the Django container
just manage createsuperuser  # Create an admin user
just prune              # Remove containers AND volumes (destructive)
```

The compose file is `docker-compose.local.yml` (set via `COMPOSE_FILE` in the justfile). Key ports: django=8000, mailpit=8025, flower=5555, node/webpack=3000.

The Django container runs `uvicorn config.asgi:application --host 0.0.0.0 --reload --reload-include '*.html'` (see `compose/local/django/start`). This means template changes trigger reload too.

### Running Tests

```bash
# Inside Docker
docker compose -f docker-compose.local.yml run django pytest
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

### Project Layout

- `config/` — Settings, root URLs, ASGI/WSGI, Celery app. Settings split: `base.py`, `local.py`, `test.py`, `production.py`.
- `config/api_router.py` — DRF router; register new API viewsets here.
- `truss/` — Application code (`APPS_DIR`). New apps go here.
- `truss/users/` — Custom user app (email-based auth, no username).
- `truss/conftest.py` — Shared pytest fixtures (`user` factory via factory_boy).

### URL Structure

| Path                 | Purpose                                                                                  |
| -------------------- | ---------------------------------------------------------------------------------------- |
| `settings.ADMIN_URL` | Django admin (default `admin/`, overridden via `DJANGO_ADMIN_URL` env var in production) |
| `/api/`              | DRF router (`config/api_router.py`)                                                      |
| `/api/users/`        | User viewset; `/api/users/me/` for current user                                          |
| `/api/auth-token/`   | Obtain DRF auth token (POST email + password)                                            |
| `/api/schema/`       | OpenAPI schema (drf-spectacular)                                                         |
| `/api/docs/`         | Swagger UI (admin-only in production)                                                    |
| `/accounts/`         | django-allauth (login, signup, email verification, MFA)                                  |
| `/users/`            | User HTML views (detail, redirect, update)                                               |

### Key Design Decisions

- **Custom User model** (`truss.users.models.User`): Email as `USERNAME_FIELD`, no username. Single `name` field. Managed by custom `UserManager`.
- **ATOMIC_REQUESTS**: Enabled in `base.py` — every view runs in a database transaction. Use `transaction.non_atomic_requests` decorator for views that need to opt out (e.g., long-running or with external API calls mid-request).
- **Authentication**: django-allauth with email-only login, mandatory email verification, MFA support.
- **API**: DRF with token + session auth. CORS restricted to `/api/*` paths.
- **Task queue**: Celery with Redis broker, django-celery-beat for periodic tasks.
- **Frontend**: Webpack (dev server on port 3000), Bootstrap 5 with custom Sass (`truss/static/sass/custom_bootstrap_vars`), django-webpack-loader.
- **Production**: Sentry (Django + Celery + Redis integrations), WhiteNoise for static files, GCS for media, Anymail/SendGrid for email.

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
