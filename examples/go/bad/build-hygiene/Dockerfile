# BAD Go builder — no WORKDIR, sudo, an unverified download, dirty apt, and a dynamic libc binary.

FROM golang:1.22 AS builder
# DF-23 no WORKDIR in either stage — everything runs from /
COPY go.mod go.sum ./
RUN go mod download
ADD ./config /build/config                       # DF-12 ADD for a local copy — use COPY
RUN apt-get update                               # DF-08 update split from install
RUN apt-get install -y git                       # DF-07 cache not cleaned · DF-09 no --no-install-recommends · DF-10 first install
RUN apt-get install -y ca-certificates           # DF-07 · DF-09 · DF-10 second separate install
RUN sudo curl -fsSL https://example.com/migrate-v1.tar.gz -o m.tar.gz \
    && tar -xzf m.tar.gz                          # DF-03 sudo in container · DF-05 download with no checksum
COPY . .
RUN go build -o /server ./cmd/server             # DF-20 no CGO_ENABLED=0 → dynamic libc, crashes on distroless/static

FROM gcr.io/distroless/static-debian12:nonroot AS runtime
COPY --from=builder --chown=nonroot:nonroot /server /server
USER nonroot
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/server", "-health-check"]
ENTRYPOINT ["/server"]
