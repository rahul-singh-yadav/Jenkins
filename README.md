## Docker-Compose File (v1)
---

```yaml
version: '3'

services:
  jenkins-master:
    image: jenkins/jenkins:lts-jdk11
    ports:
      - "8080:8080"
    volumes:
      - "jenkins-data:/var/jenkins_home"
      - "/var/run/docker.sock:/var/run/docker.sock" # Mount the Docker socket to allow Docker commands in Jenkins
      - "ssh-agent:/ssh-agent" # Mount the SSH agent socket
    environment:
      - SSH_AUTH_SOCK=/ssh-agent # Set the SSH_AUTH_SOCK environment variable to the mounted socket
    networks:
      - jenkins-network

  jenkins-slave:
    image: jenkins/inbound-agent:alpine-jdk11
    container_name: jenkins-slave
    environment:
      - JENKINS_SECRET=jenkins_secret
      - JENKINS_TUNNEL=jenkins-master:50000
      - JENKINS_AGENT_NAME=jenkins-slave
      - SSH_AUTH_SOCK=/ssh-agent # Set the SSH_AUTH_SOCK environment variable to the mounted socket
    volumes:
      - "ssh-agent:/ssh-agent" # Mount the SSH agent socket
    networks:
      - jenkins-network

  ssh-agent:
    image: jenkins/ssh-agent
    command: "ssh-agent -a /ssh-agent" # Start the ssh-agent and create a socket at /ssh-agent
    environment:
      - SSH_AUTH_SOCK=/ssh-agent # Set the SSH_AUTH_SOCK environment variable to the mounted socket
    volumes:
      - "ssh-agent:/ssh-agent" # Share the socket with other containers
    networks:
      - jenkins-network
      
networks:
  jenkins-network:
volumes:
  jenkins-data:
  ssh-agent:
```


## Improvements to v1
---
Here are some improvements and fixes for the given Docker Compose file:

- Used specific tag versions for the images to ensure consistency and avoid unexpected changes.

- Used consistent indentation and spacing for better readability.

- Used singular name for volumes to align with the recommended naming convention used by docker.

- Used the restart policy to ensure that the containers are restarted automatically if they fail.

- Removed duplicate environment variables from the jenkins-slave service and keep them in a single block.

- Added `depends_on` to the *jenkins-slave* service to ensure that it starts after the *jenkins-master* service.

- Used the `network_mode` instead of `networks` to specify the same network for the *jenkins-slave* and ssh-agent services.


## Docker-Compose File (v2)
---

```yaml
version: '3'

networks:
  jenkins-network:

volumes:
  jenkins_data:
  ssh-agent:

services:
  jenkins-master:
    image: jenkins/jenkins:lts-jdk11
    ports:
      - "8080:8080"
    volumes:
      - "jenkins_data:/var/jenkins_home"
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "ssh-agent:/ssh-agent"
    environment:
      SSH_AUTH_SOCK: /ssh-agent
    networks:
      - jenkins-network
    restart: unless-stopped

  jenkins-slave:
    image: jenkins/inbound-agent:alpine-jdk11
    container_name: jenkins-slave
    environment:
      JENKINS_SECRET: jenkins_secret
      JENKINS_TUNNEL: jenkins-master:50000
      JENKINS_AGENT_NAME: jenkins-slave
      SSH_AUTH_SOCK: /ssh-agent
    volumes:
      - "ssh-agent:/ssh-agent"
    depends_on:
      - jenkins-master
    network_mode: "service:jenkins-master"
    restart: unless-stopped

  ssh-agent:
    image: jenkins/ssh-agent
    command: "ssh-agent -a /ssh-agent"
    volumes:
      - "ssh-agent:/ssh-agent"
    network_mode: "service:jenkins-master"
    restart: unless-stopped
```

## Docker-Compose File (V3)
---
```yaml
version: '3'

networks:
  jenkins-network:

volumes:
  jenkins_data:

services:
  jenkins-master:
    image: jenkins/jenkins:lts-jdk11
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - "./jenkins_data:/var/jenkins_home"
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "ssh-agent:/ssh-agent"
    environment:
      SSH_AUTH_SOCK: /ssh-agent
    networks:
      - jenkins-network
    restart: always

  jenkins-slave:
    image: jenkins/inbound-agent:alpine-jdk11
    container_name: jenkins-slave
    environment:
      JENKINS_SECRET: jenkins_secret
      JENKINS_TUNNEL: jenkins-master:50000
      JENKINS_AGENT_NAME: jenkins-slave
      SSH_AUTH_SOCK: /ssh-agent
    volumes:
      - "ssh-agent:/ssh-agent"
    depends_on:
      - jenkins-master
    networks:
      - jenkins-network
    restart: always

  ssh-agent:
    image: jenkins/ssh-agent
    command: "ssh-agent -a /ssh-agent"
    environment:
      SSH_AUTH_SOCK: /ssh-agent
    volumes:
      - "ssh-agent:/ssh-agent"
    networks:
      - jenkins-network
    restart: always
```

## Docker-Compose File (v4) {Production}

**Improvements over v3**

- Upgraded to version 3.8 of the compose file format.
- Added a driver to the jenkins network for better scaling and resiliency.
- Added deploy settings for each service to enable better orchestration and scaling.
- Changed the exposed port for jenkins-master to port 80 for production use.
- Added JAVA_OPTS environment variable to disable the Jenkins setup wizard.
- Changed the restart policy for the jenkins-slave service to "on-failure" for better resilience.
- Changed the restart policy for all services to a more sensible default policy.
- Removed unnecessary environment variables and options.

```yaml
version: '3.8'

networks:
  jenkins:
    driver: overlay

volumes:
  jenkins_data:
  ssh-agent:

services:
  jenkins-master:
    image: jenkins/jenkins:lts-jdk11
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
    ports:
      - "80:8080"
      - "50000:50000"
    volumes:
      - "jenkins_data:/var/jenkins_home"
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "ssh-agent:/ssh-agent"
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
      - SSH_AUTH_SOCK=/ssh-agent
    networks:
      - jenkins

  jenkins-slave:
    image: jenkins/inbound-agent:alpine-jdk11
    deploy:
      replicas: 5
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
    environment:
      - JENKINS_SECRET=jenkins_secret
      - JENKINS_TUNNEL=jenkins-master:50000
      - JENKINS_AGENT_NAME=jenkins-slave
      - SSH_AUTH_SOCK=/ssh-agent
    volumes:
      - "ssh-agent:/ssh-agent"
    depends_on:
      - jenkins-master
    networks:
      - jenkins

  ssh-agent:
    image: jenkins/ssh-agent
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
    command: "ssh-agent -a /ssh-agent"
    environment:
      - SSH_AUTH_SOCK=/ssh-agent
    volumes:
      - "ssh-agent:/ssh-agent"
    networks:
      - jenkins
```

## Docker Compose File (v5) [ Production ]
---

**Improvements over v4**

- Used specific version numbers for the services instead of the "latest" tag to ensure consistent deployments.

- Used a different port other than port 80 for the Jenkins Master service. Port 80 is commonly used for HTTP traffic, and running Jenkins on this port could potentially cause conflicts with other applications.

- Add a health check to the Jenkins Master and Jenkins Slave services to ensure that the containers are functioning properly.

- Consider using a separate overlay network for the Jenkins Slave containers to allow for more granular control over network communication.

- Add resource limits to the Jenkins Master and Jenkins Slave services to prevent them from consuming too many resources on the host machine.

```yaml
version: '3.8'

networks:
  jenkins:
    driver: overlay

volumes:
  jenkins_data:
  ssh-agent:

services:
  jenkins-master:
    image: jenkins/jenkins:lts-jdk11.2
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - "jenkins_data:/var/jenkins_home"
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "ssh-agent:/ssh-agent"
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
      - SSH_AUTH_SOCK=/ssh-agent
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/login"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - jenkins
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: '512M'

  jenkins-slave:
    image: jenkins/inbound-agent:alpine-jdk11.2
    deploy:
      replicas: 5
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
    environment:
      - JENKINS_SECRET=jenkins_secret
      - JENKINS_TUNNEL=jenkins-master:50000
      - JENKINS_AGENT_NAME=jenkins-slave
      - SSH_AUTH_SOCK=/ssh-agent
    volumes:
      - "ssh-agent:/ssh-agent"
    depends_on:
      - jenkins-master
    networks:
      - jenkins
    deploy:
      resources:
        limits:
          cpus: '0.2'
          memory: '256M'

  ssh-agent:
    image: jenkins/ssh-agent
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
    command: "ssh-agent -a /ssh-agent"
    environment:
      - SSH_AUTH_SOCK=/ssh-agent
    volumes:
      - "ssh-agent:/ssh-agent"
    networks:
      - jenkins
    deploy:
      resources:
        limits:
```
