# BAD Node runtime — ships devDependencies, missing --chown, a broken healthcheck.

FROM node:20.18-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20.18-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
# DF-19 ships ALL deps incl. devDeps · DF-21 no --chown
COPY --from=builder /app/node_modules ./node_modules
# DF-21 COPY without --chown before USER app
COPY --from=builder /app/dist ./dist
USER app
EXPOSE 3000
# slim has no curl → DF-16 fires on this curl healthcheck
HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1
# DF-24 first CMD — silently ignored because a second CMD follows
CMD ["node", "dist/server.js"]
# DF-24 second CMD — only this one ever runs
CMD ["node", "dist/server.js", "--prod"]
