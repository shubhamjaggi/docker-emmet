# BAD Java image — wrong runtime base, a leaked secret, runs as root.
# `/docker-emmet examples/java/bad` reports this alongside the other two scenarios.

FROM maven:latest AS builder              # DF-15 unpinned — `latest` is a mutable pointer
WORKDIR /app
COPY . .                                  # DF-06 source copied before dependency resolution — busts mvn cache
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jdk-alpine        # DF-14 JDK in the runtime stage — a JRE is enough
WORKDIR /app
ARG DB_PASSWORD                           # DF-01 secret in ARG — recoverable if a build step writes it out
COPY --from=builder /app/target/*.jar app.jar
# Alpine ships busybox wget but NOT curl → DF-16 fires on this curl healthcheck
HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1
# no USER                                 # DF-02 runs as root
# no .dockerignore in this dir            # DF-11
CMD ["java", "-jar", "app.jar"]
