# GOOD twin of bad/runtime-and-health — correct runtime wiring, no findings.
# --chown on the COPY, exec form, a single CMD, and a healthcheck.

FROM maven:3.9.6-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml ./
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
RUN groupadd --system app && useradd --system --gid app app
# owned by the runtime user
COPY --from=builder --chown=app:app /app/target/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
# exec form, one ENTRYPOINT — the JVM is PID 1 and receives SIGTERM
ENTRYPOINT ["java", "-jar", "app.jar"]
