FROM openjdk:11 AS Build_image
RUN apt update && apt install maven -y
RUN git clone https://github.com/devopshydclub/vprofile-project.git
RUN cd vprofile-project && git checkout docker && mvn install

FROM tomcat:9-jre11
LABEL "project"="Vprofile"
LABEL "Author"="Raj"
RUN rm -rf /usr/local/tomcat/webapps/*
COPY --from=Build_image vprofile-project/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080
CMD ["catalina.sh", "run"]