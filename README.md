# TP3-Infra-Arnaud-Dubayle

# Script Terraform pour créer l'infrastructure globale 
### Script à copier/coller dans le cloudshell de AWS (ou AWS Lab) ou à coller dans un terminale connecter en SSH a AWS

```Bash
###########################################
# main.tf – Infra complète en un seul script
# ALB + Target Group + ASG + Launch Template
# Pas de TP1 -> user_data crée index.html
# Préfixes : ARN_
###########################################

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0"
    }
  }
}

provider "aws" {
  region = "eu-west-3" # Paris
}

###########################################
# Récupération VPC/Subnets par défaut
###########################################

data "aws_vpc" "default" {
  default = true
}

data "aws_subnet_ids" "default" {
  vpc_id = data.aws_vpc.default.id
}

data "aws_default_security_group" "default" {
  vpc_id = data.aws_vpc.default.id
}

###########################################
# Load Balancer (ALB)
###########################################

resource "aws_lb" "ARN_ALB" {
  name               = "ARN_ALB"
  internal           = false
  load_balancer_type = "application"
  subnets            = data.aws_subnet_ids.default.ids
  security_groups    = [data.aws_default_security_group.default.id]
}

###########################################
# Target Group
###########################################

resource "aws_lb_target_group" "ARN_TG" {
  name     = "ARN_TG"
  port     = 80
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  health_check {
    path              = "/"
    protocol          = "HTTP"
    matcher           = "200"
    interval          = 20
    timeout           = 5
    healthy_threshold = 2
    unhealthy_threshold = 2
  }
}

###########################################
# Listener HTTP
###########################################

resource "aws_lb_listener" "ARN_listener" {
  load_balancer_arn = aws_lb.ARN_ALB.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.ARN_TG.arn
  }
}

###########################################
# Launch Template
# -> installe Apache + génère index.html
###########################################

locals {
  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl enable httpd
    systemctl start httpd

    INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
    echo "<html><body><h1>Instance ID: ${INSTANCE_ID}</h1></body></html>" > /var/www/html/index.html
  EOF
}

resource "aws_launch_template" "ARN_LT" {
  name_prefix   = "ARN_LT-"
  image_id      = "ami-0c1bc246476a5572b" # AMI Amazon Linux 2 pour eu-west-3
  instance_type = "t2.micro"

  user_data = base64encode(local.user_data)

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "ARN_Instance"
    }
  }
}

###########################################
# Auto Scaling Group
###########################################

resource "aws_autoscaling_group" "ARN_ASG" {
  name                = "ARN_ASG"
  max_size            = 3
  min_size            = 1
  desired_capacity    = 1
  vpc_zone_identifier = data.aws_subnet_ids.default.ids

  launch_template {
    id      = aws_launch_template.ARN_LT.id
    version = "$Latest"
  }

  target_group_arns = [aws_lb_target_group.ARN_TG.arn]

  health_check_type         = "ELB"
  health_check_grace_period = 60

  tag {
    key                 = "Name"
    value               = "ARN_ASG_Instance"
    propagate_at_launch = true
  }

  lifecycle {
    create_before_destroy = true
  }
}

###########################################
# Outputs
###########################################

output "alb_dns_name" {
  value       = aws_lb.ARN_ALB.dns_name
  description = "DNS du load balancer"
}

output "test_url" {
  value       = "http://${aws_lb.ARN_ALB.dns_name}"
  description = "URL à ouvrir dans le navigateur"
}

```

