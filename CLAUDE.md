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

`just manage` uses `docker compose run --rm` (creates a one-off container, not exec into a running one).

Migrations run automatically on container start (`compose/local/django/start`). To create or run them manually:

```bash
just manage makemigrations
just manage migrate
```

The compose file is `docker-compose.local.yml` (set via `COMPOSE_FILE` in the justfile). Key ports: django=8000, mailpit=8025, flower=5555, node/webpack=3000.

The Django container runs `uvicorn config.asgi:application --host 0.0.0.0 --reload --reload-include '*.html'` (see `compose/local/django/start`). Template changes trigger reload too.

Debug toolbar is available in local dev at `/__debug__/`.

### Running Tests

```bash
# Inside Docker
docker compose -f docker-compose.local.yml run django pytest
docker compose -f docker-compose.local.yml run django pytest truss/users/tests/test_models.py::test_user_get_absolute_url

# With uv locally (requires local DB/Redis)
uv run pytest
uv run pytest truss/users/tests/test_views.py -k "test_detail"
```

Pytest is configured in `pyproject.toml` with `--ds=config.settings.test --reuse-db --import-mode=importlib`. Tests use `FakeWebpackLoader` (configured in `config/settings/test.py`), so the `node` container does not need to be running.

### Linting, Formatting, and Type Checking

```bash
uv run ruff check .              # Lint
uv run ruff check --fix .        # Lint with auto-fix
uv run ruff format .             # Format Python
uv run mypy truss                # Type check
uv run djlint truss/templates/   # Lint Django templates
npx prettier --check .           # Check non-Python formatting (JSON, YAML, HTML, MD)
npx prettier --write .           # Auto-fix non-Python formatting
pre-commit run --all-files       # Run all pre-commit hooks
```

Pre-commit hooks (`.pre-commit-config.yaml`): trailing-whitespace, end-of-file-fixer, check-json/toml/xml/yaml, debug-statements, check-builtin-literals, check-case-conflict, check-docstring-first, detect-private-key, django-upgrade (target 5.2). Ruff, djlint, pyproject-fmt, and Prettier are **not** pre-commit hooks — run them manually.

### CI Pipeline

CI (`.github/workflows/ci.yml`) runs on PRs and pushes to `main`:

- **Lint job**: Prettier, Ruff (check + format), pyproject-fmt, pre-commit hooks
- **Pytest job**: `makemigrations --check` (catches uncommitted migrations), then `pytest` in Docker

Ensure `pyproject-fmt` passes before pushing: `uvx pyproject-fmt pyproject.toml`.

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
- `truss/conftest.py` — Shared pytest fixtures (`user` factory via factory_boy, autouse `_media_storage` that redirects media to tmpdir).

### URL Structure

| Path                 | Purpose                                                                                  |
| -------------------- | ---------------------------------------------------------------------------------------- |
| `/`                  | Home page (`pages/home.html`)                                                            |
| `/about/`            | About page (`pages/about.html`)                                                          |
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
- **Templates**: `truss/templates/base.html` is the root layout. App templates go in `truss/templates/<app>/`. Allauth templates are overridden in `truss/templates/allauth/`.
- **Production**: Sentry (Django + Celery + Redis integrations), WhiteNoise for static files, GCS for media, Anymail/SendGrid for email.

### Adding New Apps

1. Create the app under `truss/` (e.g., `truss/myapp/`)
2. Add to `LOCAL_APPS` in `config/settings/base.py`
3. Register API viewsets in `config/api_router.py`
4. Add URL patterns in `config/urls.py`

### Adding Celery Tasks

Tasks go in each app's `tasks.py` using the `@shared_task` decorator. They are autodiscovered by `config/celery_app.py`. See `truss/users/tasks.py` for the pattern.

### Writing Tests

Tests use pytest + factory_boy. Factories live in each app's `tests/factories.py`. Follow the pattern in `truss/users/tests/factories.py`:

- Subclass `DjangoModelFactory`
- Set `django_get_or_create` on Meta to avoid duplicates
- Set `skip_postgeneration_save = True` when using `@post_generation` hooks

The `user` fixture in `truss/conftest.py` is available to all tests. Add shared fixtures there; app-specific fixtures go in the app's own `conftest.py`.

### Environment Variables

Stored in `.envs/.local/.django` and `.envs/.local/.postgres` (not committed). Key vars: `DATABASE_URL`, `REDIS_URL`, `DJANGO_SETTINGS_MODULE`, `USE_DOCKER`.

## Conventions

- **Package manager**: `uv` (not pip)
- **Linter/formatter**: Ruff for Python (see `pyproject.toml [tool.ruff]`), Prettier for everything else
- **Template linter**: djLint (profile: django)
- **Tests**: pytest + factory_boy (factories in each app's `tests/factories.py`)
- **Imports**: Ruff isort with `force-single-line = true` — one import per line
- **Django target**: 5.2 (enforced by django-upgrade pre-commit hook)
