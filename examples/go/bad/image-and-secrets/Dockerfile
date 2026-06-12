# BAD Go image — full base used as the runtime, a baked-in secret, runs as root, no healthcheck.

# DF-13 full image (ship the binary on distroless/scratch) · DF-15 unpinned
FROM golang:latest
WORKDIR /app
# DF-01 secret in ENV
ENV API_TOKEN=sk_live_9f3a8b2c
# DF-06 source copied before `go mod download` — busts the module cache
COPY . .
RUN go mod download
RUN go build -o /server ./cmd/server
# no USER                                 # DF-02 runs as root
# no HEALTHCHECK                          # DF-22 long-running server, no healthcheck
# no .dockerignore in this dir            # DF-11
EXPOSE 8080
# DF-04 shell form — sh becomes PID 1, signals lost
CMD /server
