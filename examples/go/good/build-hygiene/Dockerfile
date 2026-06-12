# GOOD twin of bad/build-hygiene — a clean builder, no findings.
# WORKDIR set, COPY not ADD, one atomic apt layer, checksum-verified download, static binary.

FROM golang:1.22-bookworm AS builder
WORKDIR /src
RUN apt-get update \
 && apt-get install -y --no-install-recommends git ca-certificates \
 && rm -rf /var/lib/apt/lists/*
COPY tool.sha256 ./
RUN curl -fsSL https://example.com/migrate-v1.tar.gz -o m.tar.gz \
 && sha256sum -c tool.sha256 \
 && tar -xzf m.tar.gz
COPY go.mod go.sum ./
RUN go mod download
COPY config ./config                       # COPY, not ADD
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot AS runtime
WORKDIR /app
COPY --from=builder --chown=nonroot:nonroot /server /server
USER nonroot
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/server", "-health-check"]
ENTRYPOINT ["/server"]
