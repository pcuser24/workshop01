---
title : "Data sources, variables and locals"
date : "`r Sys.Date()`"
weight : 411
chapter : false
--- 

## Variables
Define variables used in the project in the **variables.tf** file:
```hcl
variable "az_num" {
  type    = number
  default = 2
}

variable "namespace" {
  type    = string
  default = "ws01"
} 

variable "vpc_cidr_block" {
  type    = string
  default = "10.0.0.0/16"
}
```

## Data sources
Define queries in the **data-sources.tf** file:
```hcl
data "aws_region" "current" {}

data "aws_availability_zones" "available" {
  state = "available"
}
```

## Locals
Define local variables in the **lacals.tf** file:
```hcl
locals {
  vpc = {
    azs        = slice(data.aws_availability_zones.available.names, 0, var.az_num)
    cidr_block = var.vpc_cidr_block
  }

  s3 = {
    name = "${var.namespace}-mys3bucket"
  }

  rds = {
    allocated_storage = 20
    engine            = "PostgreSQL"
    engine_version    = "16.1"
    instance_class    = "db.t3.micro"
    db_name           = "postgres"
    username          = "postgres"
  }

  vm = {
    instance_type = "m5.large"

    instance_requirements = {
      memory_mib = {
        min = 8192
      }
      vcpu_count = {
        min = 2
      }
      instance_generations = ["current"]
    }
  }
}
```
