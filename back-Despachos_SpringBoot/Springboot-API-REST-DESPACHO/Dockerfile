FROM maven:3.8.8-eclipse-temurin-17-alpine AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S springgroup && adduser -S springuser -G springgroup
COPY --from=build /app/target/*.jar app.jar
RUN chown springuser:springgroup app.jar
USER springuser

# Configuración para el puerto 8085
EXPOSE 8085
ENTRYPOINT ["java", "-Dserver.port=8085", "-jar", "app.jar"]