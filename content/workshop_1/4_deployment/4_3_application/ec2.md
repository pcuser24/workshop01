---
title : "EC2 instances"
date : "`r Sys.Date()`"
weight : 433
chapter : false
--- 

## IAM role for the EC2 instances
Create an IAM role for the EC2 instances and attach previously created policies (Database policy and S3 policy) to it:
```hcl
data "aws_iam_policy_document" "app_assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "app_iamrole" {
  name               = "${var.namespace}-app-iamrole"
  assume_role_policy = data.aws_iam_policy_document.app_assume_role.json
  managed_policy_arns = [
    aws_iam_policy.app_db.arn,
    aws_iam_policy.app_appbucket.arn
  ]
}

resource "aws_iam_instance_profile" "app_profile" {
  name = "app-profile"
  role = aws_iam_role.app_iamrole.name
}
```

## EC2 instances
### Bastion host
For debugging purpose, create a bastion host that allows inbound SSH access from the public internet and outbound access to the private subnets.
First create a SSH key pair, save the private key to a file:
```hcl
resource "tls_private_key" "app_privatekey" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "app_keypair" {
  key_name   = "${var.namespace}-app-keypair"
  public_key = tls_private_key.app_privatekey.public_key_openssh
}

# Export the private key to a file
resource "local_file" "app_key" {
  content  = tls_private_key.app_privatekey.private_key_pem
  filename = "${path.module}/${var.namespace}-key.pem"
}
```
Create security group which allow SSH access from the public internet and outbound access to the private subnets:
```hcl
resource "aws_security_group" "bastion_sg" {
  name        = "${var.namespace}-bastion-sg"
  description = "Allow traffic to RDS instance from private subnets"
  vpc_id      = aws_vpc.ws01.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow SSH to the app instances"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = concat(values(aws_subnet.private_0)[*].cidr_block, values(aws_subnet.private_1)[*].cidr_block)
  }
}
```
Then create the bastion host:
```hcl
resource "aws_instance" "bastion" {

  lifecycle {
    prevent_destroy = false
    ignore_changes  = [iam_instance_profile, tags, tags_all]
  }
  
  ami                         = data.aws_ami.linux.id
  instance_type               = local.bastion.instance_type
  key_name                    = aws_key_pair.app_keypair.key_name
  subnet_id                   = aws_subnet.public[0].id
  vpc_security_group_ids      = [aws_security_group.bastion_sg.id]
  associate_public_ip_address = true

  tags = {
    Name = "${var.namespace}-bastion"
  }
}
```

### Launch template
Create a security group for the application instances that allows inbound traffic from the bastion host and outbound traffic to the internet:
```hcl
resource "aws_security_group" "app_sg" {
  name_prefix = "${var.namespace}-app-sg"
  vpc_id      = aws_vpc.ws01.id

  ingress {
    description = "Allow HTTP from bastion host"
    security_groups = [aws_security_group.bastion_sg.id]
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
  }

  ingress {
    description = "Allow SSH from bastion host"
    security_groups = [aws_security_group.bastion_sg.id]
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
  }

  egress {
    description      = "Allow all outbound traffic"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
  }
}
```
Create a launch template for the application instances. This will be used by the autoscaling group to launch new instances:
```hcl
resource "aws_launch_template" "apptemplate" {
  name_prefix            = "${var.namespace}-apptemplate"
  image_id               = data.aws_ami.ubuntu.id
  instance_type          = local.vm.instance_type
  key_name               = aws_key_pair.app_keypair.key_name
  vpc_security_group_ids = [aws_security_group.app_sg.id]

  iam_instance_profile {
    name = aws_iam_instance_profile.app_profile.name
  }

  monitoring {
    enabled = true
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_put_response_hop_limit = 1
    http_tokens                 = "required"
    instance_metadata_tags      = "enabled"
  }

  user_data = base64encode(templatefile("${path.module}/userdata/app_ubuntu.sh", {
    db_name            = aws_db_instance.db.db_name,
    db_username        = aws_db_instance.db.username,
    db_password        = aws_db_instance.db.password,
    db_host            = aws_db_instance.db.address,
    random_32chars_key = random_string.random_token_secret.result,
    aws_region         = data.aws_region.current.name,
    aws_s3_bucket      = aws_s3_bucket.appbucket.bucket
  }))

  update_default_version = true
}
```
{{% notice note %}}
In this script, the user data script is passed to the launch template. The user data script is a shell script that will be executed when the instance is launched. It installs the necessary packages and configures the application.
{{% /notice %}}

## Autoscaling group
### Target group
Create a target group for the application instances. This group will also be used by load balancer later on:
```hcl
resource "aws_lb_target_group" "app_lb_target_group" {
  name     = "${var.namespace}-lb-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.ws01.id

  health_check {
    healthy_threshold   = 3
    unhealthy_threshold = 10
    timeout             = 5
    interval            = 10
    path                = "/healthcheck"
    port                = 80
    protocol            = "HTTP"
  }
}
```
{{% notice info %}}
The application exposes an endpoint `/healthcheck` that returns a 200 status code. If this endpoint returns a status code other than 200 or the instance is not reachable, the instance will be marked as unhealthy.
{{% /notice %}}

### Autoscaling group
Create an autoscaling group that will launch instances based on the launch template:
```hcl
resource "aws_autoscaling_group" "app_asg" {
  name                      = "${var.namespace}-app-asg"
  min_size                  = 1
  desired_capacity          = 2
  max_size                  = 4
  health_check_grace_period = 300
  health_check_type         = "ELB"

  vpc_zone_identifier = values(aws_subnet.private_0)[*].id
  target_group_arns   = [aws_lb_target_group.app_lb_target_group.arn]

  mixed_instances_policy {
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.apptemplate.id
        version            = "$Latest"
      }
    }
  }

  lifecycle {
    create_before_destroy = true
  }

  tag {
    key                 = "Name"
    value               = "${var.namespace}-app-asg"
    propagate_at_launch = true
  }

  depends_on = [aws_db_instance.db, aws_s3_bucket.appbucket]
}
```
{{% notice note %}}
New instance will be spun up if health condition of instances in the target group is not met (e.g: HTTP request to endpoint `/healthcheck` does not return status code 200).
{{% /notice %}}

## Apply changes
1. Run `terraform validate` to validate syntax
2. Run `terraform plan` to plan the deployment
3. Run `terraform apply` to apply the deployment

## Review deployment
1. Navigate to IAM console. Verify there is `ws01-app-iamrole` IAM role which has 2 policies attached (`ws01_db_policy` and `ws01-app-appbucket`).
![IAM role](/images/ec2/ec2_iam_role01.png)
![IAM role](/images/ec2/ec2_iam_role02.png)
2. Navigate to EC2 console. Verify the bastion host is created.
![Bastion host](/images/ec2/ec2_bastion0.png)
Try SSH into the bastion host using the private key file created earlier:
![Bastion host](/images/ec2/ec2_bastion_ssh0.png)
![Bastion host](/images/ec2/ec2_bastion_ssh1.png)
3. Verify the launch template is created.
![Launch template](/images/ec2/ec2_launch_template.png)
4. Verify the security group for bastion and the application instances is created.
![Bastion security group](/images/ec2/bastion_sg.png)
![Instance security group](/images/ec2/instance_sg.png)
5. Verify the autoscaling group is created and spawn at least 1 instance having name `ws01-app-asg`.
![Autoscaling group](/images/ec2/ec2_asg.png)
1. On each `ws01-app-asg` instance, verify that IAM role `ws01-app-iamrole` is attached.
![Instance IAM role](/images/ec2/ec2_instance_iamrole.png)
