---
layout: post
title: "Configure Linux VM as a web server Nginx"
author: Sriram H. Iyer
---

## Overview

[Nginx](https://nginx.org/en/) is an open source software for web serving, reverse proxying, caching, load balancing, media streaming, and more.

In this blog, we will configure an Azure VM with Ubuntu OS as a web server using Nginx.

## Network Architecture

We will deploy an Azure Virtual Machine with Ubuntu OS for configuring Nginx as web service.

![Network Diagram](https://raw.githubusercontent.com/hisriram96/blog/2e6581f4269388ff1f98ee8d413dbebc4b4ae6e4/_pictures/azure-linux-virtual-machine-network-diagram.png)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisriram96%2Fblog%2Fmain%2F_arm-templates%2Fazure-virtual-machine-deployment.json)

## Configure web service using Nginx

#### Install Nginx service

Nginx is available in the Main repository of Canonical. No additional repository needs to be enabled for Nginx. Execute the following command to install the Nginx package.

```
sudo apt-get update
sudo apt-get install nginx
```

#### Create directory for custom website:

```
sudo mkdir /var/www/example.com
```

#### Create a landing webpage:

```
sudo vi /var/www/example.com/index.html
```

Example of the landing web page:

```
<h1>Hello World!<h1>
```

#### Create a virtual host file for your website:

```
sudo vi /etc/nginx/sites-enabled/example.com
```

Configuration of virtual host file:

```
server {
       listen 80;
       listen [::]:80;

       server_name example.com;

       root /var/www/example.com;
       index index.html;

       location / {
               try_files $uri $uri/ =404;
       }
}
```

#### Restart Nginx service:

```
sudo systemctl restart nginx
```

## Securing web service using TLS/SSL certificates

### Generate a self-signed certificate and update Nginx configuration

In this section, we will create a self-signed certificate and ```.pfx``` file using the [OpenSSL](https://www.openssl.org/) commands and configure them in Nginx.

#### Install OpenSSL

```
sudo apt-get update
sudo apt-get install openssl
```

We will create a self-signed certificate chain with own custom root CA

#### Create a key for root certificate

```
openssl ecparam -out root.key -name prime256v1 -genkey
```
   
#### Create a CSR (Certificate Signing Request) for root certificate and self-sign it

Generate the Certificate Signing Request (CSR). The CSR is a public key that is given to a CA when requesting a certificate. The CA issues the certificate for this specific request.

We will be prompted for the password for the root key, and the organizational information for the custom CA such as Country/Region, State, Org, OU, and the fully qualified domain name (this is the domain of the issuer).

```
openssl req -new -sha256 -key root.key -out root.csr
```

Create the root certificate using the root CSR. We will use this to sign your server certificate.

```
openssl x509 -req -sha256 -days 365 -in root.csr -signkey root.key -out root.crt
```

#### Create a key for server certificate 

Generate the key for the server certificate.

```
openssl ecparam -out fabrikam.key -name prime256v1 -genkey
```

#### Create the CSR for server certificate

Please note that the CN (Common Name) for the server certificate must be different from the issuer's domain. For example, in this case, the CN for the issuer is `example.com` and the server certificate's CN is `www.example.com`.

```
openssl req -new -sha256 -key server.key -out server.csr
```

Create the server certificate signing it using root key

```
openssl x509 -req -in server.csr -CA root.crt -CAkey root.key -CAcreateserial -out server.crt -days 365 -sha256
```

#### Create a full chain certificate bundling root and server certificates

````
cat server.crt > bundle.crt
cat root.crt >> bundle.crt
````

#### Install root certificate in the trust store

```
sudo apt-get install -y ca-certificates
sudo cp root.crt /usr/local/share/ca-certificates
sudo update-ca-certificates
```















