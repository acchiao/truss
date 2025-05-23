# syntax=docker/dockerfile:1

FROM node:18.15.0-bullseye-slim AS client-builder

ARG APP_HOME=/app
WORKDIR ${APP_HOME}

COPY package.json ${APP_HOME}/package.json
RUN npm install && npm cache clean --force

COPY ./ ${APP_HOME}/
RUN npm run build

# define an alias for the specfic python version used in this file.
FROM python:3.10.10-slim-bullseye AS python

# Python build stage
FROM python AS python-build-stage

ARG BUILD_ENVIRONMENT=production
ARG APP_HOME=/app
WORKDIR ${APP_HOME}

# Install apt packages
RUN apt-get update \
      && apt-get install --yes --no-install-recommends \
        build-essential \
        libpq-dev \
      && apt-get purge --yes --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
      && rm -rf /var/lib/apt/lists/*

# Requirements are installed here to ensure they will be cached.
COPY requirements/ ${APP_HOME}/

# Create Python Dependency and Sub-Dependency Wheels.
RUN pip wheel --wheel-dir /usr/src/app/wheels \
      --requirement ${BUILD_ENVIRONMENT}.txt

# Python 'run' stage
FROM python AS python-run-stage

ARG BUILD_ENVIRONMENT=production
ARG APP_HOME=/app

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV BUILD_ENV=${BUILD_ENVIRONMENT}

WORKDIR ${APP_HOME}

RUN addgroup --system django \
      && adduser --system --ingroup django django

# Install required system dependencies
RUN apt-get update \
      && apt-get install --yes --no-install-recommends \
        libpq-dev \
        gettext \
      && apt-get purge --yes --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
      && rm -rf /var/lib/apt/lists/*

# All absolute dir copies ignore workdir instruction. All relative dir copies are wrt to the workdir instruction
# copy python dependency wheels from python-build-stage
COPY --from=python-build-stage /usr/src/app/wheels /wheels/

# use wheels to install python dependencies
RUN pip install --no-cache-dir --no-index --find-links=/wheels/ /wheels/* \
      && rm -rf /wheels/

COPY --chown=django:django compose/production/django/entrypoint /entrypoint
RUN sed -i 's/\r$//g' /entrypoint \
      && chmod +x /entrypoint

COPY --chown=django:django compose/production/django/start /start
RUN sed -i 's/\r$//g' /start \
      && chmod +x /start

COPY --chown=django:django compose/production/django/celery/worker/start /start-celeryworker
RUN sed -i 's/\r$//g' /start-celeryworker \
      && chmod +x /start-celeryworker

COPY --chown=django:django compose/production/django/celery/beat/start /start-celerybeat
RUN sed -i 's/\r$//g' /start-celerybeat \
      && chmod +x /start-celerybeat

COPY --chown=django:django compose/production/django/celery/flower/start /start-flower
RUN sed -i 's/\r$//g' /start-flower \
      && chmod +x /start-flower

# copy application code to WORKDIR
COPY --from=client-builder --chown=django:django ${APP_HOME}/ ${APP_HOME}/

# make django owner of the WORKDIR directory as well.
RUN chown django:django ${APP_HOME}/

USER django

ENTRYPOINT ["/entrypoint"]
