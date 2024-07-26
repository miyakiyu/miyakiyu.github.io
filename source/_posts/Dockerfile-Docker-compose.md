---
title: Dockerfile & Docker compose
date: 2024-06-28 21:37:26
tags:
- Docker
top: 98
---
Because I’m confused by Dockerfile and Docker Compose, I have this exercise to help me understand :D
*Now, you need to create two containers (server and client) using Dockerfile and Docker Compose. The server will send “Hello, World” to the client, and after they send and receive the message three times, they will shut down.*

<!--more-->

---
My idea is to create two containers: one for the server and one for the client. We will create shell scripts for them and use netcat (nc) to send messages. Then, we will write a Dockerfile to customize the image and use Docker Compose to execute it.

This is the construct of this exercise:
```bash
.
├── client
│   ├── Dockerfile
│   └── start.sh
├── docker-compose.yml
└── server
    ├── Dockerfile
    └── start.sh
```

---
## Shell Script
We're starting with a shell script where we want to use netcat to send a message three times.
### Server
```bash
# Comfirm server is on
echo "Server on"
# For loop to send "Hello World" and pipe to 9527 port using netcat
for i in {1..3}
do
    echo "Hello World" | nc -l -p 9527
done
exit
```
### Client
```bash
# Comfirm client is on
echo "Client on"
# Receive messages from server using nc on port 9527
for i in {1..3}
do
    echo message = $(nc server 9527)
done
exit
```

---
## Dockerfile 
Dockerfile, it's writing for config docker image :
### Server 
```bash
# Pull the latest version of the image
FROM ubuntu:latest
# Renew the cache then install netcat
RUN apt-get update && apt-get install -y netcat
# Copy the shell script into image
# Copy (local) to (image)
COPY start.sh /start.sh
# Set the permission
RUN chmod +x /start.sh
# After the image start, execute the shell script within the container.
CMD["/start.sh"] 
```
### Client
Same config with server.

---
## Docker compose
We configured two services, server and client, and built them using their respective Dockerfiles located in their own directories. Both services are connected to the same network named chat to facilitate communication. The client service is configured to depend on server, ensuring that the client starts only after the server has started.

The networks section defines the Docker network, utilizing the default bridge driver. Following these configurations, we can build the Docker containers, automatically executing the shell script specified in each Dockerfile.

```config
services:
    server:
        build:
            context: ./server
        networks:
            - chat
    client:
        build:
            context: ./client
        depends_on:
            - server
        networks:
            - chat
networks:
    chat:
        driver: bridge
```
---
## Difference
So what's the difference? 

Dockerfile defines how to build a Docker image, specifying which base image to use, what software to install, and how to configure the environment within the image. I think it's like blueprint.

Docker Compose is a tool for defining and running multi-container Docker applications. It uses a configuration file to define multiple services, networks, and other configurations needed to run the application. When you use Docker Compose to start the containers, it reads this configuration to setup and interaction of multiple Docker containers according to the defined specifications.It more like manager for manage all container.


