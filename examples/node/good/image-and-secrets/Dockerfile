# GOOD twin of bad/image-and-secrets — `/docker-emmet examples/node/good` reports no issues.
# Multi-stage: build with the full toolchain, ship prod-only deps on slim as a non-root user.

FROM node:20.18-slim AS builder
WORKDIR /app
# manifest first → install layer stays cached
COPY package*.json ./
# clean, reproducible install (incl. devDeps for the build)
RUN npm ci
COPY . .
RUN npm run build

FROM node:20.18-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --chown=app:app package*.json ./
# production dependencies only
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=builder --chown=app:app /app/dist ./dist
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
# exec form — node is PID 1 and receives SIGTERM
CMD ["node", "dist/server.js"]
# DATABASE_PASSWORD injected at runtime, never baked into the image
