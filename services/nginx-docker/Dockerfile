FROM nginx:1.23.3-alpine-slim
LABEL Maintainer="rahul.singh"

## Execute shell commands
RUN apk update && \
    apk upgrade

RUN rm /etc/nginx/conf.d/default.conf

# Copy custom configuration
COPY /config/nginx.jenkins.conf /etc/nginx/conf.d/nginx.jenkins.conf
COPY /config/nginx.conf /etc/nginx/nginx.conf

# Check Config
RUN nginx -t

# Expose: HTTP
EXPOSE 80

# Run Nginx
CMD [ "nginx" ]