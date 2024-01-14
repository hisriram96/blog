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

1. Install Nginx package.

   ```
   sudo apt-get update
   sudo apt-get install nginx
   ```

2. Create directory for custom website.

   ```
   sudo mkdir /var/www/www.example.com
   ```

3. Create a landing webpage.

   ```
   sudo vi /var/www/www.example.com/index.html
   ```

   Example of the landing page.

   ```
   <h1>Hello World!<h1>
   ```

4. Create a virtual host file for your website.

   ```
   sudo vi /etc/nginx/sites-enabled/www.example.com
   ```

   Configuration of virtual host file:

   ```
   server {
          listen 80;
          listen [::]:80;

          server_name www.example.com;

          root /var/www/www.example.com;
          index index.html;

          location / {
                  try_files $uri $uri/ =404;
          }
   }
   ```

5. Restart Nginx service.

   ```
   sudo systemctl restart nginx
   ```

   Example:

   <img width="757" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/a629fef4-e607-47fe-82bc-26f3e4d3b345">

6. Verify accessing the web site.

   ```
   curl -v http://www.example.com --resolve www.example.com:80:<Public IP of the VM>
   ```

   Example:

   <img width="623" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/0a342c30-b506-44be-9f3d-ff090d2640a4">


## Securing web service using TLS/SSL certificates

In the previous sections, we configured Nginx as a web service using HTTP protocol. In order to encrypt the traffic, we need to create TLS/SSL certificate and configure Nginx to use the certificate for enabling HTTPS.

We could either create a self-signed certificate or issue a trusted CA (Certificate Authority) certificate. 

Please note that self-signed certificates are not recommended for production or in customer/public facing websites because the custom domain for which the certificate is issued is not verified. 

When a truested CA certificate is issued, the custom domain in verified by a third-party CA and is more secure.

### Generate a self-signed certificate and update Nginx configuration

In this section, we will create a self-signed SSL certificate and `.pfx` file using the [OpenSSL](https://www.openssl.org/) commands and configure them in Nginx.

> Self-signed certificates are digital certificates that are not signed by a trusted third-party CA. Self-signed certificates are created, issued, and signed by the company or developer who is responsible for the website or software being signed. This is why self-signed certificates are considered unsafe for public-facing websites and applications.

Since we will create self-signed SSL certificate with custom root CA using OpenSSL commands, we need to install OpenSSL package.

```
sudo apt-get update
sudo apt-get install openssl
```

We will create a self-signed certificate chain with own custom root CA.

1. Create a key for root certificate

   ```
   openssl ecparam -out root.key -name prime256v1 -genkey
   ```

2. Create a CSR (Certificate Signing Request) for root certificate and self-sign it

   Generate the Certificate Signing Request (CSR). The CSR is a public key that is given to a CA when requesting a certificate. The CA issues the certificate for this specific request.

   We will be prompted for the password for the root key, and the organizational information for the custom CA such as Country/Region, State, Org, OU, and the fully qualified domain name (this is the domain of the issuer).

   ```
   openssl req -new -sha256 -key root.key -out root.csr
   ```

3. Create the root certificate using the root CSR. We will use this to sign your server certificate.

   ```
   openssl x509 -req -sha256 -days 365 -in root.csr -signkey root.key -out root.crt
   ```

4. Generate the key for the server certificate.

   ```
   openssl ecparam -out server.key -name prime256v1 -genkey
   ```

5. Create the CSR for server certificate.

   > Please note that the CN (Common Name) for the server certificate must be different from the issuer's domain. For example, in this case, the CN for the issuer is `example.com` and the server certificate's CN is `www.example.com`.

   ```
   openssl req -new -sha256 -key server.key -out server.csr
   ```

6. Create the server certificate signing it using root key

   ```
   openssl x509 -req -in server.csr -CA root.crt -CAkey root.key -CAcreateserial -out server.crt -days 365 -sha256
   ```

7. Create a full chain certificate bundling root and server certificates.

   ````
   cat server.crt > bundle.crt
   cat root.crt >> bundle.crt
   ```

8. Install root certificate in the [CA trust store](https://ubuntu.com/server/docs/security-trust-store) of Ubuntu. 

   ```
   sudo apt-get install -y ca-certificates
   sudo cp root.crt /usr/local/share/ca-certificates
   sudo update-ca-certificates
   ```

### Issue a trusted CA certificate using Let's Encrypt












