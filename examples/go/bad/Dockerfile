# BAD Go service Dockerfile — `/docker-emmet examples/go/bad` shows every finding.
# Each annotation names the rule it triggers.

FROM golang:1.22 AS builder
# DF-23 no WORKDIR anywhere in the file — every COPY/RUN runs from the filesystem root
COPY go.mod go.sum ./
RUN go mod download
ADD ./config /build/config                       # DF-12 ADD for a local copy — use COPY
RUN apt-get update                               # DF-08 update split from install — stale-cache trap
RUN apt-get install -y git                       # DF-07 cache not cleaned · DF-09 no --no-install-recommends · DF-10 first of two installs
RUN apt-get install -y ca-certificates           # DF-07 · DF-09 · DF-10 second separate install — merge them
RUN sudo curl -fsSL https://example.com/migrate-v1.tar.gz -o m.tar.gz \
    && tar -xzf m.tar.gz && mv migrate /usr/local/bin/   # DF-03 sudo in container · DF-05 download with no checksum
COPY . .
RUN go build -o /server ./cmd/server             # DF-20 no CGO_ENABLED=0 → dynamic libc, crashes on distroless/static

FROM gcr.io/distroless/static AS runtime          # DF-15 untagged base — mutable pointer
COPY --from=builder /server /server              # DF-21 COPY without --chown before a non-root USER
COPY --from=builder /build/config /config        # DF-21 same — owned by root, unwritable as nonroot
USER nonroot
EXPOSE 8080
# DF-22 long-running server but no HEALTHCHECK anywhere in the file
CMD ["/server"]                                  # DF-24 this CMD is silently ignored …
CMD ["/server", "--config", "/config"]           # DF-24 … only the last CMD ever runs
# no .dockerignore in this dir                    # DF-11
