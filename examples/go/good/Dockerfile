# GOOD Go service — `/docker-emmet examples/go/good` should report no issues.
# Fully static build (no libc), shipped on distroless as a non-root user.

FROM golang:1.22-bookworm AS builder
WORKDIR /src
COPY go.mod go.sum ./                      # manifest first → module-download layer stays cached
RUN go mod download
COPY . .
# CGO disabled → a static binary that runs on scratch/distroless; -s -w strips symbols to shrink it
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot AS runtime
WORKDIR /app
COPY --from=builder --chown=nonroot:nonroot /server /server   # owned by the runtime user
USER nonroot
EXPOSE 8080
# The binary implements a health subcommand — no shell or curl needed on distroless
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/server", "-health-check"]
ENTRYPOINT ["/server"]
