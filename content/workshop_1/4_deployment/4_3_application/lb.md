---
title : "Load Balancer"
date : "`r Sys.Date()`"
weight : 434
chapter : false
--- 

## Load balancer
### Security group
Create a security group for the load balancer that allows inbound traffic on port 80 from the public internet:
```hcl
resource "aws_security_group" "app_lb_sg" {
  name_prefix = "${var.namespace}-app-lb-sg"
  vpc_id      = aws_vpc.ws01.id

  ingress {
    description = "Allow HTTP from any IP"
    cidr_blocks = ["0.0.0.0/0"]
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
  }

  ingress {
    description = "Allow HTTPS from any IP"
    cidr_blocks = ["0.0.0.0/0"]
    from_port   = 443
    to_port     = 443
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
Update the security group for the application instances to allow inbound traffic from the load balancer:
```hcl
resource "aws_security_group" "app_sg" {
  name_prefix = "${var.namespace}-app-sg"
  vpc_id      = aws_vpc.ws01.id

  ingress {
    description = "Allow HTTP from load balancer and bastion host"
    security_groups = [aws_security_group.bastion_sg.id, aws_security_group.app_lb_sg.id]
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
  }

  ...
}
```
### Application load balancer
Create a load balancer that sits on public subnet and distribute traffic to the instances in the target group:
```hcl
resource "aws_lb" "app_alb" {
  name               = "${var.namespace}-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = values(aws_subnet.public)[*].id

  tags = {
    Name = "${var.namespace}-lb"
  }

  security_groups = [aws_security_group.app_lb_sg.id]
}

resource "aws_lb_listener" "app_alb_http" {
  load_balancer_arn = aws_lb.app_alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_lb_target_group.arn
  }
}
```

## Outputs
1. Output the DNS name of the load balancer and the Cloudfront distribution:
```hcl
output "alb_endpoint_url" {
  value = "https://${aws_lb.app_alb.dns_name}"
}

output "cloudfront_endpoint_url" {
  value = "https://${aws_cloudfront_distribution.appbucket_distribution.domain_name}"
}
```

## Apply changes
1. Run `terraform validate` to validate syntax
2. Run `terraform plan` to plan the deployment
3. Run `terraform apply` to apply the deployment

## Review deployment
1. Outputs:
![Outputs](/images/outputs.png)
2. On the left pane, click on Load Balancers. Verify that the load balancer `ws01-alb` is created and correctly configured.
![Load balancer](/images/lb/lb_01.png)
![Load balancer](/images/lb/lb_02.png)
![Load balancer](/images/lb/lb_03.png)
![Load balancer](/images/lb/lb_04.png)
3. Try accessing the load balancer DNS name. Verify that the request is forwarded to the application instances.
![Load balancer](/images/lb/lb_05.png)
**The Load Balancer works, it distributed requests to instances in target groups**
4. Try creating a user by sending POST request to the load balancer DNS name:
![Load balancer](/images/lb/lb_06.png)
**The app instance is connected to S3 bucket and RDS**
5. Try accessing the avatar uploaded to S3 using DNS name of the Cloudfront distribution
![avatar](/images/lb/avatar.png)
