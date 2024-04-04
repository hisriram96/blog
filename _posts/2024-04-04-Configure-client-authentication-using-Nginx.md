---
layout: post
title: "Configure client authentication using Nginx"
author: Sriram H. Iyer
---

## Overview

In a previous [blog](https://blog.hisriram.com/2024/01/14/Configure-web-server-in-Linux-VMs-using-Nginx.html), we had deployed an Ubuntu VM in Azure and configured it as a web server using [Nginx](https://nginx.org/en/).

We also secured the web traffic by using SSL certificate.

For internal web-based applications, using SSL certificates for securing HTTPS communication in server side may not be sufficient. You would need to verify the client which is accessing the web service as well so that only authenticated clients could access it. This is called as client authentication or mutual TLS (mTLS).

We will configure Nginx so that our web server would authenticate client as well during TLS handshake in this blog.

## Pre-requisites

This blog assumes that you already went through the previous [blog](https://blog.hisriram.com/2024/01/14/Configure-web-server-in-Linux-VMs-using-Nginx.html) and have basic knowledge of Nginx configuration as we would focus on configuring mTLS using Nginx and would touch suface of basic Nginx cofnigration for web server.

## Network Architecture

We will deploy an Azure Virtual Machine with Ubuntu OS for configuring Nginx as web service.

![Network Diagram](https://raw.githubusercontent.com/hisriram96/blog/2e6581f4269388ff1f98ee8d413dbebc4b4ae6e4/_pictures/azure-linux-virtual-machine-network-diagram.png)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisriram96%2Fblog%2Fmain%2F_arm-templates%2Fazure-virtual-machine-ubuntu-deployment.json)

## Generate SSL certificate

Before we configure mTLS in Nginx, we need to issue SSL certificates and confgure web server for HTTPS.

We would create a self-signed SSL certificate using OpenSSL in this example but you should consider using SSL certificate from a valid well-known CA liek DigiCert or GoDaddy for production workloads.

```bash
sudo apt-get update
sudo apt-get install openssl
```

We will create a self-signed certificate chain with own custom root CA.

1. Create a key for root certificate

   ```bash
   openssl ecparam -out root.key -name prime256v1 -genkey
   ```

2. Create a CSR (Certificate Signing Request) for root certificate and self-sign it

   > Please note that the CN (Common Name) of the root certificate must be different from that of the server certificate. In this example, the CN for the issuer is `example.com` and the server certificate's CN is `www.example.com`.

   ```bash
   openssl req -new -sha256 -key root.key -out root.csr
   ```

3. Create the root certificate using the root CSR. We will use this to sign your server certificate.

   ```bash
   openssl x509 -req -sha256 -days 365 -in root.csr -signkey root.key -out root.crt
   ```

4. Generate the key for the server certificate.

   ```bash
   openssl ecparam -out server.key -name prime256v1 -genkey
   ```

5. Create the CSR for server certificate.

   ```bash
   openssl req -new -sha256 -key server.key -out server.csr
   ```

6. Create the server certificate signing it using root key

   ```bash
   openssl x509 -req -in server.csr -CA root.crt -CAkey root.key -CAcreateserial -out server.crt -days 365 -sha256
   ```

7. Create a full chain certificate bundling root and server certificates.

   ```bash
   cat server.crt > bundle.crt
   cat root.crt >> bundle.crt
   ```

   Example:

   <img width="721" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/39c1f863-78bd-4926-96d2-6402b007d7dd">

8. Install root certificate in the [CA trust store](https://ubuntu.com/server/docs/security-trust-store) of Ubuntu. 

   ```bash
   sudo apt-get install -y ca-certificates
   sudo cp root.crt /usr/local/share/ca-certificates
   sudo update-ca-certificates
   ```

   Example:

   <img width="543" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/e4ee3891-9504-4f38-b936-9598928ba846">

## Configure secure web service using Nginx

1. Install Nginx package.

   ```bash
   sudo apt-get update
   sudo apt-get install nginx
   ```

2. Create directory for custom website.

   ```bash
   sudo mkdir /var/www/www.example.com
   ```

3. Create a landing webpage.

   ```bash
   sudo vi /var/www/www.example.com/index.html
   ```

   Example of the landing page.

   ```bash
   <h1>Hello World!<h1>
   ```

4. Create a virtual host file for your website.

   ```bash
   sudo vi /etc/nginx/sites-enabled/www.example.com
   ```

   Contents of virtual host file `/etc/nginx/sites-enabled/www.example.com`.

   ```bash
   server {
          listen 443 ssl;
          listen [::]:443 ssl;
	   
	      ssl on;
	      ssl_certificate /home/user/bundle.crt;
	      ssl_certificate_key /home/user/server.key;
	   
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

5. Restart Nginx service.

   ```bash
   sudo systemctl restart nginx
   ```

   Example:

   <img width="757" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/a629fef4-e607-47fe-82bc-26f3e4d3b345">

6. Verify accessing the web site.

   ```bash
   curl -v http://www.example.com --resolve www.example.com:80:<Public IP of the VM>
   ```

   Example:

   <img width="623" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/0a342c30-b506-44be-9f3d-ff090d2640a4">


## Configuring client authentication or mutual TLS in Nginx

To configure client authentication in Nginx append the following code block in the virtual host file `/etc/nginx/sites-enabled/www.example.com`.

```
server {
       listen 443 ssl;
       listen [::]:443 ssl;

	   ssl_verify_client on;
	   ssl_client_certificate /home/user/bundle.crt;

       if ($ssl_client_verify != SUCCESS) { return 403; }
}
```

We enable the client authentication by setting the parameter `ssl_verify_client` to "on" and providing absolute path of the certificate (in PEM or CRT format) in the parameter "ssl_client_certificate".

If a client does not pass the certificate then this this line of code `if ($ssl_client_verify != SUCCESS) { return 403; }` dictates the server to return a 403 (Forbidden) status code.

With the above code block appeded, the virtual host file would look like below.

```
server {
    listen 80;
    server_name www.example.com;
    return 301 https://www.example.com$request_uri;
}

server {
       listen 443 ssl;
       listen [::]:443 ssl;
	   
	   ssl on;
	   ssl_certificate /home/user/bundle.crt;
	   ssl_certificate_key /home/user/server.key;
	   
       server_name www.example.com;
	   access_log /var/log/nginx/nginx.vhost.access.log;
	   error_log /var/log/nginx/nginx.vhost.error.log;
	   
       root /var/www/www.example.com;
       index index.html;
	   
       location / {
               try_files $uri $uri/ =404;
       }
}

server {
       listen 443 ssl;
       listen [::]:443 ssl;

	   ssl_verify_client on;
	   ssl_client_certificate /home/user/bundle.crt;

       if ($ssl_client_verify != SUCCESS) { return 403; }
}
```

Example:



## Verify accessing the Nginx

In case of using self-signed certificate, you would see "Not Secure" as the self-signed certificate is not secure unless the root certificate is stored in the trusted CA store of the client machine. When using `curl` command to access the web site over HTTPS, use `-k` option to proceed with TLS handshake even if not secure.

<img width="672" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/2a99e745-69cb-4df2-9bca-9c380e6b156f">

In case the trusted CA certificate is used, you could access the custom domain without using `-k` option.

<img width="612" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/52320c55-b0d1-4943-b4f8-adf717a04bb9">

<link rel="alternate" type="application/rss+xml"  href="{{ site.url }}/feed.xml" title="{{ site.title }}">
