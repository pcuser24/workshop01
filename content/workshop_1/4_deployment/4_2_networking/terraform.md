---
title : "Using Terraform"
date : "`r Sys.Date()`"
weight : 422
chapter : false
--- 

## Parameters
- Local variables for VPC and subnets:
```hcl
locals {
  vpc = {
    azs        = slice(data.aws_availability_zones.available.names, 0, var.az_num)
    cidr_block = var.vpc_cidr_block
  }
}
```

## Create resources
### VPC
```hcl
resource "aws_vpc" "ws01" {
  cidr_block           = local.vpc.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.namespace}-vpc"
  }
}
```
{{% notice note %}}
Enabling DNS support and DNS hostnames for the VPC makes instances within the VPC having public IP addresses get corresponding public DNS hostnames.
{{% /notice %}}

### Subnets

```hcl
resource "aws_subnet" "public" {
  for_each = { for index, az_name in local.vpc.azs : index => az_name }

  vpc_id                  = aws_vpc.ws01.id
  cidr_block              = cidrsubnet(aws_vpc.ws01.cidr_block, 8, (each.key + (length(local.vpc.azs) * 0)))
  availability_zone       = each.value
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.namespace}_public_subnet_${each.key}"
  }
}

resource "aws_subnet" "private_0" {
  for_each = { for index, az_name in local.vpc.azs : index => az_name }

  vpc_id                  = aws_vpc.ws01.id
  cidr_block              = cidrsubnet(aws_vpc.ws01.cidr_block, 8, (each.key + (length(local.vpc.azs) * 1)))
  availability_zone       = each.value

  tags = {
    Name = "${var.namespace}_private_subnet_0${each.key}"
  }
}

resource "aws_subnet" "private_1" {
  for_each = { for index, az_name in local.vpc.azs : index => az_name }

  vpc_id                  = aws_vpc.ws01.id
  cidr_block              = cidrsubnet(aws_vpc.ws01.cidr_block, 8, (each.key + (length(local.vpc.azs) * 2)))
  availability_zone       = each.value

  tags = {
    Name = "${var.namespace}_private_subnet_1${each.key}"
  }
}
```

### Internet Gateway

```hcl
resource "aws_internet_gateway" "default" {
  vpc_id = aws_vpc.ws01.id

  tags = {
    Name = "${var.namespace}-igw"
  }
}
```

### NAT Gateway

```hcl
resource "aws_eip" "nat_gateway" {
  tags = {
    Name = "${var.namespace}_natgw_eip"
  }
}

resource "aws_nat_gateway" "default" {
  connectivity_type = "public"
  subnet_id         = aws_subnet.public[0].id
  allocation_id     = aws_eip.nat_gateway.id
  depends_on        = [aws_internet_gateway.default]

  tags = {
    Name = "${var.namespace}_natgw"
  }
}
```

### Route Tables

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.ws01.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.default.id
  }

  tags = {
    Name = "${var.namespace}_public_rtb"
  }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.ws01.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.default.id
  }

  tags = {
    Name = "${var.namespace}_private_rtb"
  }
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private_0)

  subnet_id      = aws_subnet.private_0[count.index].id
  route_table_id = aws_route_table.private.id
}
```

## Apply changes
1. Run `terraform validate` to validate syntax
2. Run `terraform plan` to plan the deployment
3. Run `terraform apply` to apply the deployment

## Review deployment
1. Navigate to the AWS Console, search for VPC, and click on VPC to go to the VPC Console
2. On the left navigation pane, click on each of the following VPC components and verify that you have resources created that map with your Terraform deployment:

- VPC:
  ![VPC](/images/networking/tf_vpc.png)
- Subnets:
  ![Subnets](/images/networking/tf_subnets.png)
- Route Tables:
  ![Public Route Tables](/images/networking/tf_public_rtb.png)
  ![Private Route Tables](/images/networking/tf_private_rtb.png)
- Internet Gateway:
  ![Internet Gateway](/images/networking/tf_igw.png)
- NAT Gateway:
  ![NAT Gateway](/images/networking/tf_natgw.png)
