FROM openjdk:8-jdk


WORKDIR /usr/src/app
COPY *.jar ./app.jar


ENTRYPOINT ["java","-jar","./app.jar", "--port=80"]
