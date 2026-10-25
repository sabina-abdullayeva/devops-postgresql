# Полная сборка с нуля — для docker build / docker compose без CI
FROM node:22-slim AS base


FROM base AS build
SHELL [ "/bin/sh", "-eu", "-c" ]
ENV NEXT_TELEMETRY_DISABLED="1"

WORKDIR /build
COPY package.json package-lock.json ./
RUN npm config set fetch-retries 5 \
  && npm config set fetch-retry-mintimeout 20000 \
  && npm config set fetch-retry-maxtimeout 120000 \
  && npm ci --no-audit --no-fund

COPY src/ src/
COPY public/ public/
COPY next.config.mjs tsconfig.json ./
RUN \
  npx next build \
  && cp --recursive .next/standalone /app \
  && cp --recursive public /app/public \
  && cp --recursive .next/static /app/.next/static


FROM base AS final

WORKDIR /app
ENV NODE_ENV="production"
ENV NEXT_TELEMETRY_DISABLED="1"
ENV HOSTNAME="0.0.0.0"
ENV PORT="8080"

COPY --from=build --chown=node:node /app/ ./

USER node
EXPOSE ${PORT}/tcp
ENTRYPOINT [ "node", "/app/server.js" ]
