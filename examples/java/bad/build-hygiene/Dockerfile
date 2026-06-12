# BAD Java builder — messy build hygiene: no WORKDIR, sudo, an unverified
# download, and split/dirty apt layers. The runtime stage here is fine.

FROM maven:3.9.6-eclipse-temurin-21 AS builder
# DF-23 no WORKDIR in either stage — every COPY/RUN runs from the filesystem root
# DF-12 ADD for a local copy — use COPY
ADD ./config /build/config
# DF-08 update split from install — stale-cache trap
RUN apt-get update
# DF-07 cache not cleaned · DF-09 no --no-install-recommends · DF-10 first install
RUN apt-get install -y git
# DF-07 · DF-09 · DF-10 second separate install — merge into one
RUN apt-get install -y unzip
# DF-03 sudo in container · DF-05 download with no checksum
RUN sudo curl -fsSL https://example.com/grpc_health_probe -o /bin/probe \
    && chmod +x /bin/probe
# DF-06 full context copied before the build
COPY . .
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-jammy AS runtime
COPY --from=builder --chown=1000:1000 /target/*.jar /app.jar
USER 1000
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "/app.jar"]
