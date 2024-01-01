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

## DNS configuration scenarios with BIND9

Before we proceed with configuration of DNS server using BIND9, we need to understand the different types of DNS servers. We will focus on caching nameserver and authoratative nameserver.

A _caching nameserver_ does not host any domain but it will recursively resolve the DNS name queries and remember the answer when the domain is queried again.

An _authoratative nameserver_ contains a zone file and is authoritative for that zone. Hence, DNS name queries are answered by the authoratative nameserver itself.

We will configure BIND9 as both caching and authoratative nameserver.

## Prerequisites

Since we will be deploying resouces in Azure in this blog, we will need an Azure subscription.

## Network Architecture

We will deploy a Virtual Network consisting of two Virtual Machines as DNS servers.

These primary and secondary DNS servers will host an internal DNS zone "example.internal" and will also act as name caching servers for all other domains.

We also have another VM configured as web server which we will try to access from another VM within the VNet using FQDN "example.internal".

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

   It is a best practice to install ```bind9utils``` package for verifying configuration of BIND9. This package provides number utilities like ```named-checkconfig``` and ```named-checkzone``` which would indicate any errors in the configuration.

   ```
   sudo apt-get install bind9utils
   ```

   You may also want to install ```dnsutils``` package for DNS utilities like ```dig``` or ```nslookup```.

   ```
   sudo apt-get install dnsutils
   ```

   Example:

   <img width="500" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/b3fff4df-a5c5-41cd-8592-37cae90446dc">

2. Configure Azure VM as caching nameserver using BIND9.

   The default configuration of BIND9 acts as a caching server. We need to edit the ```/etc/bind/named.conf.options``` file using editors like ```vi```, ```nano```, ```code```, etc.  to set the IP addresses of Azure recursive resolver.

   ```
   sudo vi /etc/bind/named.conf.options
   ```

   ```
   // Create an ACL (Access Control List) to allow name queries from trusted networks
   acl "trusted" {
           10.0.0.0/16; // IP address prefix of the Virtual Network 
    };
   
   options {
           directory "/var/cache/bind";
   
           // If there is a firewall between you and nameservers you want
           // to talk to, you may need to fix the firewall to allow multiple
           // ports to talk.  See http://www.kb.cert.org/vuls/id/800113
   
           // If your ISP provided one or more IP addresses for stable
           // nameservers, you probably want to use them as forwarders.
           // Uncomment the following block, and insert the addresses replacing
           // the all-0's placeholder.
   
           allow-query { any; }; // Allow any name query
           allow-query-cache { trusted; }; // Allow trusted networks specified in the ACL to query the nameserver for non-authoritative data such as recursive queries
           allow-recursion { trusted; }; // Act as a recursive server. Although recursion is yes by default so this line could also be deleted

           // Specify a list of IP addresses of nameservers to which the name queries should be forwarded for recursive resolution
           forwarders {
                   168.63.129.16; // This is the recursive resolver IP address of Azure
           };
   
           //========================================================================
           // If BIND logs error messages about the root key being expired,
           // you will need to update your keys.  See https://www.isc.org/bind-keys
           //========================================================================
           dnssec-validation auto;
           listen-on { any; };
           listen-on-v6 { any; };
   };
   ```

   Example:

   <img width="994" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/22ed03e5-317b-47cf-b918-0995c58b0297">

   If the ```bind9utils``` was installed in previous step, then we could verify if our configuration of ```/etc/bind/named.conf.options``` has any errors by executing ```named-checkconf``` command.

   ```
   named-checkconf -p /etc/bind/named.conf.options
   ```

   The ```-p``` option prints the configuration if no errors were detected.

   Example:

   <img width="513" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/d850ad83-05ab-4713-a28f-814d796a7e5c">





