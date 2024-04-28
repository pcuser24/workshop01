---
title : "Application"
date : "`r Sys.Date()`"
weight : 43
chapter : true
pre : " <b> 4.3. </b> "
--- 

# Deployment

This chapter focus on deploying main components of the application: EC2 instances, RDS database, S3 bucket, and CloudFront distribution. To secure the application, IAM roles and policies are created and attached to the resources. Furthurmore, to ensure high availability and fault tolerance, the resources are deployed across multiple availability zones and the application is load balanced using an Application Load Balancer while static content is served using CloudFront distribution.
