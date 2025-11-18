# Development build for Spring Boot application with hot reload
FROM maven:3.9-eclipse-temurin-17
WORKDIR /app

# Install curl for healthcheck
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Copy pom.xml and download dependencies (for better caching)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Expose port
EXPOSE 8080

# Run with Spring Boot DevTools for hot reload
ENTRYPOINT ["mvn", "spring-boot:run"]
