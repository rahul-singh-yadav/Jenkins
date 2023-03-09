# Slim based image
FROM jenkins/jenkins:2.375.3-lts-alpine
LABEL maintainer="rahul.yadav@****.com"

# CPU Tuning
ENV JAVA_OPTS="-Xmx4096m"

# Connection Pools Tuning & Log File Setup
ENV JENKINS_OPTS="--handlerCountMax=50 --logfile=/var/log/jenkins/jenkins.log"


