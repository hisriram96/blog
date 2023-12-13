---
layout: post
title: "Configure IPsec VPN in an Ubuntu VM using strongSwan"
author: Sriram
---

## What is strongSwan?

strongSwan is open-source package for IPsec VPN. You could use strongSwan for configuring VPN in Linux, UNIX, and BSD Operating Systems.

In this example, we will configure an IPsec VPN between two Ubuntu VMs. However, same configuration can be applied for RedHat and other distos.

Please note that the configuration is done on two Azure VMs which are in different VNets and there is no connecitivity between VNets in any manner. So, before proceeding with rest of the Blog, please create two Azure VMs in different VNets with public IPs. You could perform this excerise in AWS as well by creating two EC2 instances in different VPCs

## Network Architecture

Before we configure IPsec VPN using strongSwan, we need to deploy Azure VMs with Public IPs which are in different VNets and there is no connecitivity between VNets in any manner. You could perform this excerise in AWS as well by creating two EC2 instances in different VPCs



## Enable IP forwarding

Before configuring IPsec VPN, we must make sure that our VMs act as routers.

Router can accept and forward packets which are not destined to itself. This behaviour of router is called *IP forwarding*. However, servers/VMs would discard the packets which have destination IP differnt from the its own.

This is default behaviour for servers/VMs and we can change it. We can configure the OS of the server/VM to accept the packets which have destination IP differnt from the its own interface IP. This configuration differs depending on the OS.

In Ubuntu distros, you could enable IP forwarding by modifying the contents of ```/etc/sysctl.conf```. One of the easy way to do it is by using ```sed``` command as below.

```
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
sudo sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/g' /etc/sysctl.conf
sudo sysctl -p
```

Since we deployed Azure VMs, we must also enable IP forwarding in the vNIC of the Azure VMs.

Please note that enabling IP forwarding in the vNIC of the Azure VM is an additional step as it does does not enable IP forwarding in the guest OS so we still need to enable IP forwarding in Ubuntu OS.

## Configuring strongSwan

With our network infrastructure ready and IP forwarding enabled in the OS and in VNIC, we could proceed with the configuration of IPsec VPN.

1. Install strongSwan.

   ```
   sudo apt-get update
   sudo apt-get install strongswan -y
   sudo apt-get install strongswan-pki -y
   sudo apt-get install libstrongswan-extra-plugins -y
   ```

2. Configure IPsec VPN by editing the ipsec.conf file.

   ```
   sudo vi /etc/ipsec.conf
   ```

   Contents of the ```ipsec.conf``` file.

   ```
	config setup
		charondebug="all"
		uniqueids=yes
	conn tunnel21
		type=tunnel
		left=<Private_IP_address_of_the_VM>
		leftsubnet=<Local_IP_prefix>
		right=<VPN_peer_IP_address>
		rightsubnet=<Remote_IP_prefix>
		keyexchange=ikev2
		keyingtries=%forever
		authby=psk
		ike=aes256-sha256-modp1024!
		esp=aes256-sha256!
		keyingtries=%forever
		auto=start
		dpdaction=restart
		dpddelay=45s
		dpdtimeout=45s
		ikelifetime=28800s
		lifetime=27000s
		lifebytes=102400000
   ```

5. Configure pre-shared key for VPN in ipsec.secrets file.

   ```
   sudo vi /etc/ipsec.secrets
   ```

   Contents of the ```ipsec.secrets``` file.

   ```
   <Private_IP_address_of_the_VM> <VPN_peer_IP_address> : PSK "<pre-shared_key>"
   ```

   Please make sure that the same pre-shared key is configured on both IPsec peers. Otherwise, VPN will be down.

7. Restart the strongSwan process.

   ```
   sudo systemctl restart ipsec
   sudo systemctl status ipsec
   ```

<link rel="alternate" type="application/rss+xml"  href="{{ site.url }}/feed.xml" title="{{ site.title }}">
