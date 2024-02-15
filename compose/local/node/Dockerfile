# syntax=docker/dockerfile:1

FROM node:21.6.1-bullseye-slim

WORKDIR /app

COPY package.json /app/package.json

RUN npm install && npm cache clean --force

ENV PATH node_modules/.bin/:$PATH
