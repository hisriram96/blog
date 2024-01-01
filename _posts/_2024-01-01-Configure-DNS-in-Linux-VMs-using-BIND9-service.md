---
layout: post
title: "Configure DNS in Linux VMs using BIND9 service"
author: Sriram H. Iyer
---

## Overview

Each and every device in computer network require an IP address for communication. Imagine the tediousness of typing IP addresses for connecting to servers hosting the websites.

Glad that we do not have to do that because of DNS. 

DNS (Domain Name System) is used to resolve (translate) domain names to IP addresses and vice versa. A DNS server, also known as a nameserver, maps IP addresses to hostnames or domain names.

BIND (Berkley Internet Naming Daemon) is an open source software package for implementing DNS servers for a number of Linux and Unix based operating systems.

In this blog, we will deploy an Azure Virtual Machine with Ubuntu 22.04 LTS and configure DNS using [BIND9](https://bind9.readthedocs.io/en/v9.18.21/).

## Prerequisites

Since we will be deploying resouces in Azure in this blog, we will need an Azure subscription.

## Network Architecture

We will deploy a Virtual Network consisting of two Virtual Machines. We will configure one of the Virtual Machines as DNS server using BIND9 in following sections.

![Network Diagram](https://raw.githubusercontent.com/hisriram96/blog/main/_pictures/azure-dns-lab-two-virtual-machines-network-diagram.png)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fgithub.com%2Fhisriram96%2Fblog%2Fblob%2F6f71eb3e8290ed8e88edf87df5d822835c04ab0e%2F_arm-templates%2Fazure-virtual-ubuntu-machine-dns-deployment.json)

## Configure DNS server using BIND9

1. Install BIND9 service.

   ```
   sudo apt-get update
   sudo apt-get install bind9
   ```

   You could optionally install the BIND9 documentation (very useful).

   ```
   sudo apt-get install bind9-doc
   ```

   You may also want to install ```dnsutils``` package for DNS utilities like ```dig``` or ```nslookup```.

   ```
   sudo apt-get install dnsutils
   ```

2. Configure Azure VM as caching nameserver using BIND9.



