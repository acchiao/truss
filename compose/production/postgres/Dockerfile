# syntax=docker/dockerfile:1

FROM postgres:16.3-bullseye

COPY compose/production/postgres/maintenance /usr/local/bin/maintenance

RUN chmod +x /usr/local/bin/maintenance/*

RUN mv /usr/local/bin/maintenance/* /usr/local/bin \
      && rmdir /usr/local/bin/maintenance
