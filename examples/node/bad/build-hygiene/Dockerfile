# BAD Node builder — no WORKDIR, sudo, an unverified download, dirty apt layers.
# The runtime stage here is fine; the mistakes are all build-side.

FROM node:20.18-slim AS builder
# DF-23 no WORKDIR in either stage — everything runs from /
ADD ./assets /build/assets                       # DF-12 ADD for a local copy — use COPY
RUN apt-get update                               # DF-08 update split from install
RUN apt-get install -y git                       # DF-07 cache not cleaned · DF-09 no --no-install-recommends · DF-10 first install
RUN apt-get install -y python3                    # DF-07 · DF-09 · DF-10 second separate install — merge them
RUN sudo curl -fsSL https://example.com/tool.tar.gz -o tool.tar.gz \
    && tar -xzf tool.tar.gz                       # DF-03 sudo in container · DF-05 download with no checksum
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20.18-slim AS runtime
COPY --from=builder --chown=node:node /dist ./dist
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/server.js"]
