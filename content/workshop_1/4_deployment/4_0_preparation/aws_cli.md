---
title : "AWS CLI"
date : "`r Sys.Date()`"
weight : 401
chapter : false
--- 

## AWS CLI
- Install the AWS CLI to interact with AWS services from the command line.
- To configure credentials, run and follow the prompts:
  ```bash
  aws configure
  ```
- To verify the configuration:
  ```bash
  aws sts get-caller-identity
  ```
