# GOOD twin of bad/build-hygiene — a clean builder, no findings.
# WORKDIR set, COPY not ADD, one atomic apt layer, checksum-verified download, no sudo.

FROM maven:3.9.6-eclipse-temurin-21 AS builder
WORKDIR /app
RUN apt-get update \
 && apt-get install -y --no-install-recommends git unzip \
 && rm -rf /var/lib/apt/lists/*
COPY probe.sha256 ./
RUN curl -fsSL https://example.com/grpc_health_probe -o /usr/local/bin/probe \
 && sha256sum -c probe.sha256 \
 && chmod +x /usr/local/bin/probe
COPY config ./config                       # COPY, not ADD — explicit, no surprise tar-extraction
COPY pom.xml ./
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
RUN groupadd --system app && useradd --system --gid app app
COPY --from=builder --chown=app:app /app/target/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
