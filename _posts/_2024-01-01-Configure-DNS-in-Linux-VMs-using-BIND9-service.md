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
