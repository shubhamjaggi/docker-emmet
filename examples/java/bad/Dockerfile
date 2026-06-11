# BAD Java/Spring Boot Dockerfile — `/docker-emmet examples/java/bad`.

FROM maven:latest AS builder              # DF-15 unpinned
WORKDIR /app
COPY . .                                  # DF-06 source copied before dependency resolution
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jdk-alpine        # DF-14 JDK in runtime — a JRE is enough
WORKDIR /app
ARG DB_PASSWORD                           # DF-01 secret in ARG
COPY --from=builder /app/target/*.jar app.jar

# Alpine ships busybox wget but NOT curl → DF-16 fires on this curl healthcheck
HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1

# no USER                                 # DF-02 runs as root
# no .dockerignore                        # DF-11
CMD ["java", "-jar", "app.jar"]
