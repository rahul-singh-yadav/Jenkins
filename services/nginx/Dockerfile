FROM nginx:1.23.3-alpine-slim
LABEL Maintainer="rahul.singh"

## Execute shell commands
RUN apk apt update && \
    apk upgrade
RUN mkdir -p /home/backup
# RUN mv /etc/nginx/conf.d/default.conf /home/backup
RUN mv /etc/nginx/nginx.conf /home/backup

# Copy custom configuration
COPY $pwd/nginx-docker/config/nginx.jenkins.conf /etc/nginx/conf.d/nginx.jenkins.conf
COPY $pwd/nginx-docker/config/nginx.conf /etc/nginx/nginx.conf

# Check Cofig
RUN nginx -t

# Expose: HTTP
EXPOSE 80

# Run Nginx
CMD [ "nginx" ]
