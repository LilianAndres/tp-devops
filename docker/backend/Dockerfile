# Build stage
# Source image with a name alias
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build

# Create env variable
ENV MYAPP_HOME /opt/myapp

# Set the workdir path
WORKDIR $MYAPP_HOME

# Copy files from source to destination on the container
COPY pom.xml .
COPY src ./src

# Run mvn package command to build the app
RUN mvn package -DskipTests

# Run stage
# Source image
FROM amazoncorretto:17

# Create env variable
ENV MYAPP_HOME /opt/myapp

# Set the workdir path
WORKDIR $MYAPP_HOME

# Copy the compiled code from the previous stage into the new one as myapp.jar
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# Run the JAR and start the backend
ENTRYPOINT java -jar myapp.jar