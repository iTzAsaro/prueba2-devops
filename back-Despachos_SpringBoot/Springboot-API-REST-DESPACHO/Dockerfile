# Etapa 1: build (compila el JAR usando Maven)
FROM maven:3.9.9-eclipse-temurin-17 AS build
WORKDIR /app
# Copia solo el pom para habilitar cache de dependencias
COPY pom.xml ./
# Descarga dependencias una vez (mejora tiempos de build)
RUN mvn -B -q -DskipTests dependency:go-offline
# Copia el código y empaqueta el JAR
COPY src ./src
# Limpieza: deja solo el JAR final y elimina cache/artefactos para reducir tamaño
RUN mvn -B -DskipTests package && \
    cp target/*.jar /app/app.jar && \
    rm -rf /root/.m2/repository target

# Etapa 2: runtime (solo JRE + el JAR, sin toolchain de build)
FROM eclipse-temurin:17.0.19_10-jre-alpine-3.23 AS runtime
# Usuario no-root (mínimo privilegio) y dependencias mínimas para healthcheck HTTP
RUN addgroup -S app -g 10001 && adduser -S app -G app -u 10001 && \
    apk add --no-cache ca-certificates curl
WORKDIR /app
# Copia únicamente el JAR desde la etapa de build
COPY --from=build /app/app.jar /app/app.jar
# Ajusta permisos para ejecutar como usuario sin privilegios
RUN chown -R app:app /app
USER app
EXPOSE 8081
# Parametrización opcional de la JVM (heap, GC, etc.) sin rebuild (JVM lo lee automáticamente)
ENV JAVA_TOOL_OPTIONS=""
ENTRYPOINT ["java","-jar","/app/app.jar"]
