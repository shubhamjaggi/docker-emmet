# GOOD app image — multi-stage, prod-only dependencies, non-root, exec form, healthcheck.
# `/docker-emmet examples/insecure/good` should report no issues.

FROM node:20.18-slim AS builder
WORKDIR /app
COPY package*.json ./                      # manifest first → install layer stays cached
RUN npm ci                                 # clean, reproducible install (incl. devDeps for the build)
COPY . .
RUN npm run build

FROM node:20.18-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --chown=app:app package*.json ./
RUN npm ci --omit=dev && npm cache clean --force         # production dependencies only
COPY --from=builder --chown=app:app /app/dist ./dist
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/server.js"]             # exec form — node is PID 1 and receives SIGTERM
