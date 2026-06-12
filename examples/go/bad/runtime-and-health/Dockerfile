# BAD Go runtime — missing --chown, a healthcheck calling a missing binary, duplicate CMD.

FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /server ./cmd/server

FROM gcr.io/distroless/base-debian12 AS runtime
COPY --from=builder /server /server              # DF-21 COPY without --chown before a non-root USER
USER nonroot
EXPOSE 8080
# distroless has no shell or wget → DF-16 fires on this wget healthcheck
HEALTHCHECK CMD wget -qO- http://localhost:8080/health || exit 1
CMD ["/server"]                                  # DF-24 with the next line — two CMD
CMD ["/server", "--verbose"]                     # DF-24 only the last CMD runs
