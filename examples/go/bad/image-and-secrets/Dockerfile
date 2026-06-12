# BAD Go image — full base used as the runtime, a baked-in secret, runs as root, no healthcheck.

FROM golang:latest                        # DF-13 full image (ship the binary on distroless/scratch) · DF-15 unpinned
WORKDIR /app
ENV API_TOKEN=sk_live_9f3a8b2c            # DF-01 secret in ENV
COPY . .                                  # DF-06 source copied before `go mod download` — busts the module cache
RUN go mod download
RUN go build -o /server ./cmd/server
# no USER                                 # DF-02 runs as root
# no HEALTHCHECK                          # DF-22 long-running server, no healthcheck
# no .dockerignore in this dir            # DF-11
EXPOSE 8080
CMD /server                               # DF-04 shell form — sh becomes PID 1, signals lost
