FROM eclipse-temurin:17-jre
# Set working directory 
WORKDIR /app

# Expose backend port 
EXPOSE 8080

# Copy compiled jar 
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar

# Start application 
ENTRYPOINT ["java","-jar","app.jar"]


