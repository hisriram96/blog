---
layout: post
title: "Containerize a Nginx web application using Docker"
author: Sriram H. Iyer
---

## Overview

Imagine you have spent much of your valuable time in developing an application and if works as expected in your system. However, you later find out that the same application does not work as expected for other users.

This issue is quite common and is caused due to multiple factors such as the required libraries, frameworks, and other dependencies are missing in the systems of others users.

Containerization solves this problem by packaging your application code and all of its dependencies so that it runs in an isolated environment called as container.

Containers can be moved across platforms and infrastructures, the application would work in same way because it has everything it needs to run successfully within it.

In order to containerize an application, we need help of container runtimes like [Podman](https://www.redhat.com/en/topics/containers/what-is-podman) and [Docker](https://www.docker.com/products/container-runtime/).

In this blog we will use Docker to create a container running [Nginx](https://nginx.org/en/) web service.

## Pre-requisites

Install [Docker Engine](https://docs.docker.com/engine/install/) or [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/).

## Create a Docker container

The creation of a Docker container is 3 step process:
1. Create a Dockerfile.
2. Build an image.
3. Run a container.

A Dockerfile is a script consisting of a set of instructions for building a Docker image.

A Docker image is a collection of packaged application, along with its dependencies, and information on what processes it runs when launched.

A Docker container is a running instance of the Docker image. Multiple containers can be created from the image but an image cannot be created from a container.

### Create a Dockerfile

Create a file named `examplefile.dockerfile` with contents below. This file is your Docker file.

```docker
FROM ubuntu:latest

RUN <<EOT bash
apt-get update && apt-get install nginx openssl -y
openssl ecparam -out root.key -name prime256v1 -genkey
openssl req -new -sha256 -key root.key -out root.csr -subj "/C=IN/ST=TL/L=HYD/O=myOrg/OU=IT/CN=example.com/emailAddress=hostmaster@example.com"
openssl x509 -req -sha256 -days 365 -in root.csr -signkey root.key -out root.crt
openssl ecparam -out server.key -name prime256v1 -genkey
openssl req -new -sha256 -key server.key -out server.csr -subj "/C=IN/ST=TL/L=HYD/O=myOrg/OU=IT/CN=www.example.com/emailAddress=hostmaster@example.com"
openssl x509 -req -in server.csr -CA root.crt -CAkey root.key -CAcreateserial -out server.crt -days 365 -sha256
cat server.crt >> bundle.crt && cat root.crt >> bundle.crt
mv server.key /etc/ssl/private/server.key
mv bundle.crt /etc/ssl/certs/bundle.crt
mkdir -p /var/www/www.example.com
touch /var/www/www.example.com/index.html
echo '<h1>Hello World!</h1>' >> /var/www/www.example.com/index.html
touch /var/log/nginx/nginx.vhost.access.log
touch /var/log/nginx/nginx.vhost.error.log
EOT

COPY example.conf /etc/nginx/sites-enabled/example.conf

EXPOSE 80
EXPOSE 443

STOPSIGNAL SIGQUIT

ENTRYPOINT ["nginx", "-g", "daemon off;"]

```

Let's explore and understand the instructions used in our Dockerfile.

The `FROM` instruction lets us to specify our base image for your application. A base image is an OS image on top of which we will configure our application and its dependencies to bouild our own image. There are many images available in the DockerHub which can be used as a base image. In this example, we are using an Ubuntu image.

The `RUN` instruction lets us declare the commands which we want to execute for configuring our applications and its dependencies. We could have multiple `RUN` instructions in a Dockerfile but we have used [here-document](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_07_04) for running multiple commands delimited by `EOT` instead.

The commands used are for installing Nginx and OpenSSL, generating self-signed SSL certificates, and creating a landing page (index.html) for our website.

The `COPY` instruction is used to copy the Nginx configuration file from our local system to the filesystem of the container.

The `EXPOSE` instruction lets the specified ports 80 and 443 to be in listening state at runtime of the container.

The `STOPSIGNAL` instruction lets us to set the system call signal that will be sent to the container when it is stopped exit. We use the SIGQUIT system call signal to gracefully stop the Nginx process when the container exits.

The `ENTRYPOINT` instruction lets us to specify the command which we want to execite when we run the container. The command `nginx -g 'daemon off;'` is for nginx to stay in the foreground, so that Docker can track the process properly (otherwise your container will stop immediately after starting).

Example:

<img width="995" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/cb4be2ea-fab5-4a32-806d-58ceac44b7f7">

Create the Nginx configuration file named `example.conf` with contents below. Please make sure that the `example.conf` and `examplefile.dockerfile` are in same directory.

```
server {
        listen 80;
        server_name www.example.com;
        return 301 https://www.example.com$request_uri;
}

server {
        listen 443 ssl;
        listen [::]:443 ssl;

        ssl_certificate /etc/ssl/certs/bundle.crt;
        ssl_certificate_key /etc/ssl/private/server.key;

        server_name www.example.com;
        access_log /var/log/nginx/nginx.vhost.access.log;
        error_log /var/log/nginx/nginx.vhost.error.log;

        root /var/www/www.example.com;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
}

```

### Build a Docker image

Execute the command below to build a Docker image.

```
docker build -f examplefile.dockerfile -t exampleimage .
```

> We are copying the Nginx configuration file `example.conf` from our local filesystem to the filesystem of container for which the configuration file `example.conf` must be in the same directory as the Dockerfile `examplefile.dockerfile` as we have specific context as our current working directory `.`.

Example:

<img width="1180" alt="image-1" src="https://github.com/hisriram96/blog/assets/56336513/12748bf8-eda2-445e-882c-9b9ea034721c">

### Run a Docker container

Execute the command below to run a Docker container.

```
docker run -d -p 80:80 -p 443:443 --name examplecontainer exampleimage
```

> Although we have specified the ports 80 and 443 in the Dockerfile, the ports actually remain in listening state only in the container and in the host where the container is running. For opening the ports in the host, we use the `-p <hostport>:<containerport>` option to map the host port to container port.

Execute the `docker ps` command to verify if the container starts without any issue.

Example:

<img width="1175" alt="image-2" src="https://github.com/hisriram96/blog/assets/56336513/21396ac1-b571-4e99-9c62-eb833d5c7107">

## Verify accessing the Nginx

We will use the `curl` utility to verify if the web page is accessible from the host. We should be able to see the "Hello World" web page which we have configured in our Nginx container.

<img width="599" alt="image-3" src="https://github.com/hisriram96/blog/assets/56336513/1d1f9262-52aa-4a9f-a06c-2cad2a80afe7">

<img width="587" alt="image-4" src="https://github.com/hisriram96/blog/assets/56336513/00b0301f-f071-43b1-9333-f12c45eb5bd3">
