# Etapa 1: build (compila el JAR usando Maven)
FROM maven:3.9.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml ./
RUN mvn -B -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -B -DskipTests package && \
    cp target/*.jar /app/app.jar && \
    rm -rf /root/.m2/repository target

# Etapa 2: runtime (solo JRE + el JAR, sin toolchain de build)
FROM eclipse-temurin:17.0.19_10-jre-alpine-3.23 AS runtime
RUN addgroup -S app -g 10001 && adduser -S app -G app -u 10001 && \
    apk add --no-cache ca-certificates curl
WORKDIR /app
COPY --from=build /app/app.jar /app/app.jar
RUN chown -R app:app /app
USER app
EXPOSE 8080
ENV JAVA_TOOL_OPTIONS=""
ENTRYPOINT ["java","-jar","/app/app.jar"]
