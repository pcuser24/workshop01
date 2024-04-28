---
title : "RDS"
date : "`r Sys.Date()`"
weight : 431
chapter : false
--- 

## Local variables
Local variables for RDS instance:
```hcl
locals {
  ...
  rds = {
    allocated_storage = 20
    engine            = "PostgreSQL"
    engine_version    = "16.1"
    instance_class    = "db.t3.micro"
    db_name           = "postgres"
    username          = "postgres"
  }
}
```

## Resources
### Secrets
Store and rotate secrets using Secrets Manager:
```hcl
resource "random_password" "default" {
  length           = 32
  special          = false
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

resource "aws_secretsmanager_secret" "db" {
  name_prefix             = "${var.namespace}_db-secrets"
  description             = "Password to the RDS"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.default.result
}
```

### Subnet Group
Create subnet groups where RDS instances will reside:
```hcl
resource "aws_db_subnet_group" "dbgroup" {
  name       = "${var.namespace}_db-subnetgroup"
  subnet_ids = values(aws_subnet.private_1)[*].id

  tags = {
    Name = "${var.namespace}_db-subnetgroup"
  }
}
```

### Security
The RDS should only be accessible by app instances from the private subnets. To achieve the effect, create a security group for the RDS instance:
```hcl
resource "aws_security_group" "db" {
  name        = "${var.namespace}_db-sg"
  description = "Allow traffic to RDS instance from private subnets"
  vpc_id      = aws_vpc.ws01.id

  ingress {
    from_port   = local.rds.port
    to_port     = local.rds.port
    protocol    = "tcp"
    cidr_blocks = concat(values(aws_subnet.private_0)[*].cidr_block, values(aws_subnet.private_1)[*].cidr_block)
  }

  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
}
```

### DB instances
Now, create the primary DB instance:
```hcl
resource "aws_db_instance" "db" {
  identifier = "${var.namespace}-db"

  allocated_storage       = local.rds.allocated_storage
  engine                  = local.rds.engine
  engine_version          = local.rds.engine_version
  instance_class          = local.rds.instance_class
  db_name                 = local.rds.db_name
  username                = local.rds.username
  password                = aws_secretsmanager_secret_version.db.secret_string
  db_subnet_group_name    = aws_db_subnet_group.dbgroup.name
  vpc_security_group_ids  = [aws_security_group.db.id]
  multi_az                = true
  skip_final_snapshot     = true
  backup_retention_period = 7

  tags = {
    Name = "${var.namespace}-db"
  }
}
```

Create a read replica for the primary DB instance:
```hcl
resource "aws_db_instance" "replica-db" {
  identifier          = "${var.namespace}-db-replica"
  instance_class      = local.rds.instance_class
  skip_final_snapshot = true
  backup_retention_period = 7
  replicate_source_db = aws_db_instance.db.identifier
}
```

{{% notice note %}}
To enable read replicas, the primary DB instance must have backups enabled. 'backup_retention_period' is set to 7 days in this example to retain backups for 7 days. The primary instance cannot be temporarily stopped if read replicas are enabled.
{{% /notice %}}

### IAM policy
- Create an IAM policy that allows the EC2 instances to read and write to the RDS instance:
```hcl
resource "aws_iam_policy" "app_db" {
  lifecycle {
    create_before_destroy = false
  }
  
  name        = "${var.namespace}_db_policy"
  description = "Allow all actions on RDS instance"
  policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Sid" : "AllowAllActions",
        "Effect" : "Allow",
        "Action" : [
          "rds:*"
        ],
        "Resource" : "${aws_db_instance.db.arn}"
      }
    ]
  })
}

```

## Apply changes
1. Run `terraform init` to install new `random` provider
2. Run `terraform validate` to validate syntax
3. Run `terraform plan` to plan the deployment
4. Run `terraform apply` to apply the deployment

## Review deployment
1. Navigate to RDS console
2. Verify the primary and replica instances are created
![RDS primary instance and replica](/images/rds/rds_01.png)
3. On the left pane, navigate to `Subnet Group` tab. Verify the subnet group is created
![RDS subnet group](/images/rds/rds_sg.png)
4. Go to VPC console, click on Security Group and search for `ws01_db-sg`.Verify the security group is correctly configured
![RDS security group](/images/rds/rds_sg0.png)
![RDS security group](/images/rds/rds_sg1.png)
5. Go to VPC console, click on Security Group and search for `ws01_db_policy`. Verify the IAM policy is created with sufficient permissions
![RDS IAM policy](/images/rds/rds_iam_policy.png)
6. Go to AWS Secrets Manager console to view all available secrets. Search for `ws01_db-secrets` and verify that there is a secret for database password
![RDS secrets](/images/rds/rds_secrets.png)
