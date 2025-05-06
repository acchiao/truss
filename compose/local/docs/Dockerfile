# syntax=docker/dockerfile:1

FROM python:3.10.10-slim-bullseye AS python

# Python build stage
FROM python AS python-build-stage

ENV PYTHONDONTWRITEBYTECODE=1

RUN apt-get update \
      && apt-get install --yes --no-install-recommends \
        build-essential \
        libpq-dev \
      && apt-get purge --yes --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
      && rm -rf /var/lib/apt/lists/*

# Requirements are installed here to ensure they will be cached.
COPY requirements/ /requirements/

# create python dependency wheels
RUN pip wheel \
      --no-cache-dir \
      --wheel-dir /usr/src/app/wheels \
      --requirement /requirements/local.txt \
      --requirement /requirements/production.txt \
    && rm -rf /requirements

# Python 'run' stage
FROM python AS python-run-stage

ARG BUILD_ENVIRONMENT
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

RUN apt-get update \
      && apt-get install --yes --no-install-recommends \
        make \
        libpq-dev \
        gettext \
        texlive-latex-recommended \
        texlive-fonts-recommended \
        texlive-latex-extra \
        latexmk \
      && apt-get purge --yes --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
      && rm -rf /var/lib/apt/lists/*

# copy python dependency wheels from python-build-stage
COPY --from=python-build-stage /usr/src/app/wheels/ /wheels/

# use wheels to install python dependencies
RUN pip install --no-cache-dir /wheels/* \
      && rm -rf /wheels

COPY compose/local/docs/start /start-docs
RUN sed -i 's/\r$//g' /start-docs \
      && chmod +x /start-docs

WORKDIR /docs
