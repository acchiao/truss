# define an alias for the specific python version used in this file.
FROM ghcr.io/astral-sh/uv:python3.13-bookworm-slim AS python

# Python build stage
FROM python AS python-build-stage

ARG APP_HOME=/app

WORKDIR ${APP_HOME}

# we need to move the virtualenv outside of the $APP_HOME directory because it will be overriden by the docker compose mount
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=0

# Install apt packages: build deps plus devcontainer utils, in a single layer.
# Cache mounts keep downloaded .debs and package lists across rebuilds, so a
# `just prune` full rebuild does not re-download everything.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    rm -f /etc/apt/apt.conf.d/docker-clean \
    && apt-get update && apt-get install --no-install-recommends -y \
      # dependencies for building Python packages
      build-essential \
      # psycopg dependencies
      libpq-dev \
      gettext \
      wait-for-it \
      # devcontainer dependencies and utils
      sudo git bash-completion nano ssh

# Create devcontainer user and add it to sudoers
RUN groupadd --gid 1000 dev-user \
  && useradd --uid 1000 --gid dev-user --shell /bin/bash --create-home dev-user \
  && echo dev-user ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/dev-user \
  && chmod 0440 /etc/sudoers.d/dev-user

ENV PATH="/${APP_HOME}/.venv/bin:$PATH"
ENV PYTHONPATH="${APP_HOME}/.venv/lib/python3.13/site-packages:$PYTHONPATH"

# Install dependencies first (cached unless pyproject.toml/uv.lock change).
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    --mount=type=bind,source=uv.lock,target=uv.lock:rw \
    uv sync --no-install-project

# Entrypoint and start scripts: stable, so copy before the source tree to keep
# them out of the cache-busting COPY below.
COPY ./compose/production/django/entrypoint /entrypoint
COPY ./compose/local/django/start /start
COPY ./compose/local/django/celery/worker/start /start-celeryworker
COPY ./compose/local/django/celery/beat/start /start-celerybeat
COPY ./compose/local/django/celery/flower/start /start-flower
RUN sed -i 's/\r$//g' /entrypoint /start /start-celeryworker /start-celerybeat /start-flower \
    && chmod +x /entrypoint /start /start-celeryworker /start-celerybeat /start-flower

COPY . ${APP_HOME}

# Install the project itself (source changes only invalidate from here down).
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    --mount=type=bind,source=uv.lock,target=uv.lock:rw \
    uv sync

ENTRYPOINT ["/entrypoint"]
