# BAD app image for the insecure stack — ships dev dependencies into production.
# `/docker-emmet examples/insecure/bad` reports these alongside the compose findings.

FROM node:20.18-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                                 # devDependencies are needed to build…
COPY . .
RUN npm run build

FROM node:20.18-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder /app/node_modules ./node_modules   # DF-19 ships ALL deps incl. devDeps · DF-21 no --chown
COPY --from=builder /app/dist ./dist                   # DF-21 COPY without --chown before USER app
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/server.js"]
# no .dockerignore in this dir               # DF-11
