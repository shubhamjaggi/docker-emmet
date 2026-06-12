# GOOD twin of bad/runtime-and-health — --chown, a binary-based healthcheck, a single CMD.

FROM golang:1.22-bookworm AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot AS runtime
WORKDIR /app
# owned by the runtime user
COPY --from=builder --chown=nonroot:nonroot /server /server
USER nonroot
EXPOSE 8080
# The binary implements a health subcommand — no shell or wget needed on distroless
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/server", "-health-check"]
ENTRYPOINT ["/server"]
