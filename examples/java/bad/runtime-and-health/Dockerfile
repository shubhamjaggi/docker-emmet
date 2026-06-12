# BAD Java runtime stage — permission and signal-handling mistakes.
# The build stage is clean; the problems are all in how the runtime is wired up.

FROM maven:3.9.6-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml ./
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
RUN groupadd --system app && useradd --system --gid app app
# DF-21 COPY without --chown before a non-root USER → unwritable as `app`
COPY --from=builder /app/target/*.jar app.jar
USER app
EXPOSE 8080
# DF-22 long-running server but no HEALTHCHECK anywhere in the file
# DF-04 shell form — sh becomes PID 1, SIGTERM never reaches the JVM
CMD java -jar app.jar
# DF-24 second CMD — only this one runs, the shell-form CMD above is silently ignored
CMD ["java", "-jar", "app.jar", "--debug"]
