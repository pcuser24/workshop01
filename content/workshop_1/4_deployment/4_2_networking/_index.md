---
title : "Networking Infrastructure"
date : "`r Sys.Date()`"
weight : 42
chapter : true
pre : " <b> 4.2. </b> "
--- 

# Networking infrastructure

With the IAM resources successfully created on the previous page, we can now continue with the network resources we will need to host our application.

![Networking infrastructure](/images/networking.png)
## Networking infrastructure
- The VPC has a CIDR block of `10.0.0.0/16`.
- The application is deployed on 2 Availability Zones (AZs) in the same region to ensure high availability.
- The public subnets are where the resources that need to be accessed from the internet reside, such as NAT Gateway, Load Balancer, and Bastion Host.
- The private subnets serve as the backend for the application, where the application instances and database are placed. Resources inside these subnets must be strictly secured and cannot be accessed from the internet directly.
