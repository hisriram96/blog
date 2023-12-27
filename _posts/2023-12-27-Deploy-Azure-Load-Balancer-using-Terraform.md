---
layout: post
title: "Deploy Azure Load Balancer using Terraform"
author: Sriram H. Iyer
---

## Overview

[Azure Load Balancer](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview) is used for distributing load (incoming traffic) to multiple resources hosting the same application. This ensures high availability of the application as the Load Balancers distributes traffic to healthy resources.

There are two types of Azure Load Balancers:
1. Public Load Balancer.
2. Private or Internal Load Balancer.

Public Load Balancer has public IP address as its frontend and is used for load balancing traffic sourcing from internet. An Internal Load Balancer has private IP address as its frontend and is used for load balancing traffic to internal applications from private networks.

In this blog, we will deploy an Azure Load Balancer for distributing traffic to two Azure Virtual Machines hosting a simple Nginx web service using Terraform.

Please note that this blog assumes that you know how to deploy of Azure Load Balancers using Azure Portal and you have an Azure Subscription.

## Prerequisites

Please install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) and [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) for deploying Azure resources.

## Deploy a Public Load Balancer

In this section, We will deploy a public load balancer and two Azure virtual machines with a custom script extension to configure the Nginx web service and host a simple web page. We will perform complete deployment using Terraform.

### Network Diagram

This is how our architecture will look after the deployment is completed.

![Network Diagram](https://raw.githubusercontent.com/hisriram96/blog/main/_pictures/azure-public-load-balancer-with-two-virtual-machines-network-diagram.png)

### Create and Deploy Terraform script

1. Create a directory and make it as your current directory.

   ```
   mkdir load-balancer-demo
   cd load-balancer-demo
   ```
   
2. Create a file named ```providers.tf``` and paste the configuration below. Here we have configured ```azurerm``` as Terraform provider for creating and managing our Azure resources.

   ```
   terraform {
     required_version = ">=0.12"
   
     required_providers {
       azapi = {
         source  = "azure/azapi"
         version = "~>1.5"
       }
       azurerm = {
         source  = "hashicorp/azurerm"
         version = "~>2.0"
       }
       random = {
         source  = "hashicorp/random"
         version = "~>3.0"
       }
     }
   }
   
   provider "azurerm" {
     features {}
   }
   ```

3. Create a file named ```variables.tf``` and paste the configuration below. We declare all the variables that we intend to use in our Terraform deployment in the ```variables.tf``` file. You could modify the default values as per your choice or naming convention for Azure resources.

   ```
   variable "resource_group_location" {
     type        = string
     default     = "eastus"
     description = "Location of the resource group."
   }
   
   variable "resource_group_name" {
     type        = string
     default     = "test-group"
     description = "Name of the resource group."
   }
   
   variable "username" {
     type        = string
     default     = "microsoft"
     description = "The username for the local account that will be created on the new VM."
   }
   
   variable "password" {
     type        = string
     default     = "Microsoft@123"
     description = "The passoword for the local account that will be created on the new VM."
   }
   
   variable "virtual_network_name" {
     type        = string
     default     = "test-vnet"
     description = "Name of the Virtual Network."
   }
   
   variable "subnet_name" {
     type        = string
     default     = "test-subnet"
     description = "Name of the subnet."
   }
   
   variable public_ip_name {
     type        = string
     default     = "test-public-ip"
     description = "Name of the Public IP."
   }
   
   variable network_security_group_name {
     type        = string
     default     = "test-nsg"
     description = "Name of the Network Security Group."
   }
   
   variable "network_interface_name" {
     type        = string
     default     = "test-nic"
     description = "Name of the Network Interface."  
   }
   
   variable "virtual_machine_name" {
     type        = string
     default     = "test-vm"
     description = "Name of the Virtual Machine."
   }
   
   variable "virtual_machine_size" {
     type        = string
     default     = "Standard_B2s"
     description = "Size or SKU of the Virtual Machine."
   }
   
   variable "disk_name" {
     type        = string
     default     = "test-disk"
     description = "Name of the OS disk of the Virtual Machine."
   }
   
   variable "redundancy_type" {
     type        = string
     default     = "Standard_LRS"
     description = "Storage redundancy type of the OS disk."
   }
   
   variable "load_balancer_name" {
     type        = string
     default     = "test-lb"
     description = "Name of the Load Balancer."
   }
   ```

3. Create a file named ```main.tf``` and paste the configuration below. The ```main.tf``` is our configuration file where we use to deploy our Azure resources.

   ```
   #Create Resource Group
   resource "azurerm_resource_group" "my_resource_group" {
     location = var.resource_group_location
     name     = var.resource_group_name
   }
   
   # Create Virtual Network
   resource "azurerm_virtual_network" "my_virtual_network" {
     name                = var.virtual_network_name
     address_space       = ["10.0.0.0/16"]
     location            = azurerm_resource_group.my_resource_group.location
     resource_group_name = azurerm_resource_group.my_resource_group.name
   }
   
   # Create a subnet in the Virtual Network
   resource "azurerm_subnet" "my_subnet" {
     name                 = var.subnet_name
     resource_group_name  = azurerm_resource_group.my_resource_group.name
     virtual_network_name = azurerm_virtual_network.my_virtual_network.name
     address_prefixes     = ["10.0.1.0/24"]
   }
   
   # Create Network Security Group and rules
   resource "azurerm_network_security_group" "my_nsg" {
     name                = var.network_security_group_name
     location            = azurerm_resource_group.my_resource_group.location
     resource_group_name = azurerm_resource_group.my_resource_group.name
   
     security_rule {
       name                       = "web"
       priority                   = 1008
       direction                  = "Inbound"
       access                     = "Allow"
       protocol                   = "Tcp"
       source_port_range          = "*"
       destination_port_range     = "80"
       source_address_prefix      = "*"
       destination_address_prefix = "10.0.1.0/24"
     }
   }
   
   # Associate the Network Security Group to the subnet
   resource "azurerm_subnet_network_security_group_association" "my_nsg_association" {
     subnet_id                 = azurerm_subnet.my_subnet.id
     network_security_group_id = azurerm_network_security_group.my_nsg.id
   }
   
   # Create Public IP
   resource "azurerm_public_ip" "my_public_ip" {
     name                = var.public_ip_name
     location            = azurerm_resource_group.my_resource_group.location
     resource_group_name = azurerm_resource_group.my_resource_group.name
     allocation_method   = "Static"
     sku                 = "Standard"
   }
   
   # Create Network Interface
   resource "azurerm_network_interface" "my_nic" {
     count               = 2
     name                = "${var.network_interface_name}${count.index}"
     location            = azurerm_resource_group.my_resource_group.location
     resource_group_name = azurerm_resource_group.my_resource_group.name
   
     ip_configuration {
       name                          = "ipconfig${count.index}"
       subnet_id                     = azurerm_subnet.my_subnet.id
       private_ip_address_allocation = "Dynamic"
       primary = true
     }
   }
   
   # Associate Network Interface to the Backend Pool of the Load Balancer
   resource "azurerm_network_interface_backend_address_pool_association" "my_nic_lb_pool" {
     count                   = 2
     network_interface_id    = azurerm_network_interface.my_nic[count.index].id
     ip_configuration_name   = "ipconfig${count.index}"
     backend_address_pool_id = azurerm_lb_backend_address_pool.my_lb_pool.id
   }
   
   # Create Virtual Machine
   resource "azurerm_linux_virtual_machine" "my_vm" {
     count                 = 2
     name                  = "${var.virtual_machine_name}${count.index}"
     location              = azurerm_resource_group.my_resource_group.location
     resource_group_name   = azurerm_resource_group.my_resource_group.name
     network_interface_ids = [azurerm_network_interface.my_nic[count.index].id]
     size                  = var.virtual_machine_size
   
     os_disk {
       name                 = "${var.disk_name}${count.index}"
       caching              = "ReadWrite"
       storage_account_type = var.redundancy_type
     }
   
     source_image_reference {
       publisher = "Canonical"
       offer     = "0001-com-ubuntu-server-jammy"
       sku       = "22_04-lts-gen2"
       version   = "latest"
     }
   
     admin_username                  = var.username
     admin_password                  = var.password
     disable_password_authentication = false
   
   }
   
   # Enable virtual machine extension and install Nginx
   resource "azurerm_virtual_machine_extension" "my_vm_extension" {
     count                = 2
     name                 = "Nginx"
     virtual_machine_id   = azurerm_linux_virtual_machine.my_vm[count.index].id
     publisher            = "Microsoft.Azure.Extensions"
     type                 = "CustomScript"
     type_handler_version = "2.0"
   
     settings = <<SETTINGS
    {
     "commandToExecute": "sudo apt-get update && sudo apt-get install nginx -y && echo '<h1>Hello World</h1>' > /var/www/html/index.html && sudo systemctl restart nginx"
    }
   SETTINGS
   
   }
   
   # Create Public Load Balancer
   resource "azurerm_lb" "my_lb" {
     name                = var.load_balancer_name
     location            = azurerm_resource_group.my_resource_group.location
     resource_group_name = azurerm_resource_group.my_resource_group.name
     sku                 = "Standard"
   
     frontend_ip_configuration {
       name                 = var.public_ip_name
       public_ip_address_id = azurerm_public_ip.my_public_ip.id
     }
   }
   
   resource "azurerm_lb_backend_address_pool" "my_lb_pool" {
     loadbalancer_id      = azurerm_lb.my_lb.id
     name                 = "test-pool"
   }
   
   resource "azurerm_lb_probe" "my_lb_probe" {
     resource_group_name = azurerm_resource_group.my_resource_group.name
     loadbalancer_id     = azurerm_lb.my_lb.id
     name                = "test-probe"
     port                = 80
   }
   
   resource "azurerm_lb_rule" "my_lb_rule" {
     resource_group_name            = azurerm_resource_group.my_resource_group.name
     loadbalancer_id                = azurerm_lb.my_lb.id
     name                           = "test-rule"
     protocol                       = "Tcp"
     frontend_port                  = 80
     backend_port                   = 80
     disable_outbound_snat          = true
     frontend_ip_configuration_name = var.public_ip_name
     probe_id                       = azurerm_lb_probe.my_lb_probe.id
     backend_address_pool_ids       = [azurerm_lb_backend_address_pool.my_lb_pool.id]
   }
   
   resource "azurerm_lb_outbound_rule" "my_lboutbound_rule" {
     resource_group_name     = azurerm_resource_group.my_resource_group.name
     name                    = "test-outbound"
     loadbalancer_id         = azurerm_lb.my_lb.id
     protocol                = "Tcp"
     backend_address_pool_id = azurerm_lb_backend_address_pool.my_lb_pool.id
   
     frontend_ip_configuration {
       name = var.public_ip_name
     }
   }
   ```

4. Create a file named ```outputs.tf``` and paste the configuration below. This is display the IP address in URL format which we could use for accessing our application.

   ```
   output "public_ip_address" {
     value = "http://${azurerm_public_ip.my_public_ip.ip_address}"
   }
   ```

5. Initialize the working directory containing Terraform configuration files (```load-balancer-demo``` in our case).

   ```
   terraform init -upgrade
   ```

6. Create an execution plan to preview the Terraform deployment.

   ```
   terraform plan -out main.tfplan
   ```

7. Apply Terraform configuration previewed in the execution plan.

   ```
   terraform apply main.tfplan
   ```

### Verify the deployment

After the ```terraform apply``` command is executed successfully, you can verify if you could access our web page using the public IP displayed in the output ```terraform apply``` command.

<img width="302" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/f02b30f8-389f-4cd9-b4bb-f9717d1aaeb3">

### Delete the resources

In order to avoid any extra charges, it is advisable to delete the resources which are not required. You could delete all the Azure resouces which we have deployed so far using Azure Portal or by executing the following Terraform commands.

   ```
   terraform plan -destroy -out main.destroy.tfplan
   terraform apply main.destroy.tfplan
   ```

Please note that the above commands need to be executed in the working directory containing Terraform configuration files (```load-balancer-demo``` in our case).




