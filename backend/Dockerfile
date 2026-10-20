# Imagen base liviana optimizada para Java 17 en produccion
FROM eclipse-temurin:17-jre-alpine

# Metadatos del microservicio
LABEL maintainer="IasLab - Universidad Icesi"
LABEL service="product-api"
LABEL course="Ingenieria de Software V"

WORKDIR /app

# Copia el binario empaquetado por Maven
COPY target/product-api-1.0.4.jar app.jar

# Expone el puerto del microservicio
EXPOSE 8080

# Comando de ejecucion optimizado para contenedores
ENTRYPOINT ["java", "-jar", "app.jar"]


FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# Wildcard: no depende de la version exacta del JAR (1.0.0, 1.0.0-SNAPSHOT, etc.)
COPY target/product-api-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]