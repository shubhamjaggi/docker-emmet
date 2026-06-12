# GOOD twin of bad/runtime-and-health — prod-only deps, --chown, working healthcheck, single CMD.

FROM node:20.18-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20.18-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --chown=app:app package*.json ./
RUN npm ci --omit=dev && npm cache clean --force         # no devDependencies in the runtime image
# owned by the runtime user
COPY --from=builder --chown=app:app /app/dist ./dist
USER app
EXPOSE 3000
# Node's built-in fetch — no curl/wget needed on slim
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/server.js"]
