# GOOD twin of bad/build-hygiene — a clean builder, no findings.
# WORKDIR set, COPY not ADD, one atomic apt layer, checksum-verified download, no sudo.

FROM node:20.18-slim AS builder
WORKDIR /app
RUN apt-get update \
 && apt-get install -y --no-install-recommends git python3 \
 && rm -rf /var/lib/apt/lists/*
COPY tool.sha256 ./
RUN curl -fsSL https://example.com/tool.tar.gz -o tool.tar.gz \
 && sha256sum -c tool.sha256 \
 && tar -xzf tool.tar.gz
COPY package*.json ./
RUN npm ci
COPY assets ./assets                       # COPY, not ADD — explicit, no surprise tar-extraction
COPY . .
RUN npm run build

FROM node:20.18-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder --chown=app:app /app/dist ./dist
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/server.js"]
