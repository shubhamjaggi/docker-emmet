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
COPY --from=builder /app/node_modules ./node_modules   # DF-19 ships ALL deps incl. devDeps · DF-21 no --chown
COPY --from=builder /app/dist ./dist                   # DF-21 COPY without --chown before USER app
USER app
EXPOSE 3000
# slim has no curl → DF-16 fires on this curl healthcheck
HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]                         # DF-24 with the next line — two CMD
CMD ["node", "dist/server.js", "--prod"]               # DF-24 only the last CMD ever runs
