# GOOD Java/Spring Boot Dockerfile — `/docker-emmet examples/java/good` should report no issues.
# Build with the full JDK + Maven, ship the JAR on a JRE-only Alpine image as a non-root user.

FROM maven:3.9.6-eclipse-temurin-21-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q          # resolve deps before source → cached layer
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder --chown=app:app /app/target/*.jar app.jar
USER app
EXPOSE 8080
# Alpine's busybox ships wget, so this probe works without installing anything
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
# Container-aware heap sizing prevents OOM kills; exec form makes the JVM PID 1
ENTRYPOINT ["java", \
            "-XX:+UseContainerSupport", \
            "-XX:MaxRAMPercentage=75.0", \
            "-XX:+ExitOnOutOfMemoryError", \
            "-jar", "app.jar"]
# DB credentials injected at runtime via SPRING_DATASOURCE_PASSWORD, never baked in
