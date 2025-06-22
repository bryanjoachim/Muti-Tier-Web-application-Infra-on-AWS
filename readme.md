
# Multi-Tier Web Application Infrastructure Using Terraform



## Create a Ec2 instance
Create a Ec2 instance config (Amazon linux 2023) t2.micro. Install httpd server and mysql client. Once created stop this ec2 instance and generate an Ami out on it.

## Add/edit EC2 user policies
Add "AmazonEC2FullAccess" , "AmazonRDSFullAccess" and "AmazonVPCFullAccess"


## Create provider.tf file
```bash
provider "aws" {
  region = "us-east-1"
}
```

## Create output.tf
```bash
output "vpc_id" {
  value = aws_vpc.main.id
}
```

## Create variable.tf
```bash
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "key_name" {
  default = "my-ec2-key" 
}
```

## Create main.tf
Make sure to copy the AMI ID to app servers or else it wont work
```bash
resource "aws_vpc" "main" {
    cidr_block = var.vpc_cidr

}

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private_1" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "us-east-1a"
}

resource "aws_subnet" "private_2" {
    vpc_id = aws_vpc.main.id
    cidr_block = "10.0.3.0/24"
    availability_zone = "us-east-1b"
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "publicRoute" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "publicAssoc" {
  subnet_id = aws_subnet.public.id
  route_table_id = aws_route_table.publicRoute.id
}

resource "aws_db_subnet_group" "rdsGroup" {
  name = "rds subnet group"
  subnet_ids = [aws_subnet.private_1.id, aws_subnet.private_2.id]
}

resource "aws_security_group" "web_sg" {
  name = "web_sg"
  description = "Allow Http and  https"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  }

  ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  }


  ingress {
    from_port = 443
    to_port = 443
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "app_sg" {
  name = "app_sg"
  description = "Allow from web tier"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port = 0
    to_port = 65535
    protocol = "tcp"
    security_groups = [aws_security_group.web_sg.id]

  }

   ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    security_groups = [aws_security_group.web_sg.id]

  }
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    security_groups = [aws_security_group.alb_sg.id]

  }

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "alb_sg" {
    name = "alb_sg"
    description = "Allow Http inbound"
    vpc_id = aws_vpc.main.id
     
     ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
    
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

}

resource "aws_instance" "app_server_1" {
  ami               = "Copy the AMI id of ec2 instance"
  instance_type     = "t2.micro"
  subnet_id         = aws_subnet.private_1.id
  security_groups   = [aws_security_group.app_sg.id]
  key_name          = "my-ec2-key"
  iam_instance_profile = aws_iam_instance_profile.app_instance_profile.name
}
resource "aws_instance" "app_server_2" {
  ami = "Copy the AMI id of ec2 instance"
  instance_type     = "t2.micro"
  subnet_id         = aws_subnet.private_2.id
  security_groups   = [aws_security_group.app_sg.id]
  key_name          = "my-ec2-key" 
  iam_instance_profile = aws_iam_instance_profile.app_instance_profile.name

  
}

resource "aws_instance" "bastion" {
  ami                         = "ami-02457590d33d576c3" 
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public.id
  associate_public_ip_address = true
  key_name                    = "my-ec2-key" 
  security_groups = [aws_security_group.web_sg.id]

  
}


resource "aws_lb" "app_alb" {
    name = "app-alb"
    internal = "true"
    load_balancer_type = "application"
    subnets = [aws_subnet.private_1.id, aws_subnet.private_2.id]
    security_groups = [aws_security_group.alb_sg.id]

}

resource "aws_lb_target_group" "app_tg" {
    name = "app-target-group"
    port = 80
    protocol = "HTTP"
    vpc_id = aws_vpc.main.id
    
    health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200"
  }
}


#rds security group
resource "aws_security_group" "rds_sg" {
    name = "rds_sg"
    description = "Allow traffic from app servers only"
    vpc_id = aws_vpc.main.id

    ingress {
        from_port = 3306
        to_port = 3306
        protocol = "tcp"
        security_groups = [aws_security_group.app_sg.id]
    }

    egress {
        from_port = 0
        to_port = 0
        protocol = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
  
}

resource "aws_lb_listener" "app_listener" {
    load_balancer_arn = aws_lb.app_alb.arn
    port = 80
    protocol = "HTTP"
    default_action {
      type = "forward"
      target_group_arn = aws_lb_target_group.app_tg.arn
    }

}


resource "aws_lb_target_group_attachment" "app1" {

  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id = aws_instance.app_server_1.id
  port = 80
}

resource "aws_lb_target_group_attachment" "app2" {

  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id = aws_instance.app_server_2.id
  port = 80
}


#Rds Instance

resource "aws_db_instance" "rdsDb" {
  identifier = "app-db"
  engine = "mysql"
  instance_class = "db.t3.micro"
  allocated_storage = 20
  username = "admin"
  password = "Goodness"
  db_name = "appdb"
  publicly_accessible = false
  skip_final_snapshot = true

  backup_retention_period = 7
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  db_subnet_group_name = aws_db_subnet_group.rdsGroup.name

}


resource "aws_iam_role" "app_role" {
  name = "app-instance-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "app_attach" {
  role       = aws_iam_role.app_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonRDSFullAccess"
}

resource "aws_iam_instance_profile" "app_instance_profile" {
  name = "app-instance-profile"
  role = aws_iam_role.app_role.name
}

# Launch Template for App Instances
resource "aws_launch_template" "app_lt" {
  name_prefix   = "app-lt-"
  image_id      = "ami-08e3fb120c98716a2" # Use your actual AMI ID
  instance_type = "t2.micro"
  key_name      = "my-ec2-key"

  iam_instance_profile {
    name = aws_iam_instance_profile.app_instance_profile.name
  }

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.app_sg.id]
  }

  tag_specifications {
    resource_type = "instance"

    tags = {
      Name = "AppInstance"
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app_asg" {
  name                      = "app-asg"
  min_size                  = 2
  max_size                  = 5
  desired_capacity          = 2
  vpc_zone_identifier       = [aws_subnet.private_1.id, aws_subnet.private_2.id]
  launch_template {
    id      = aws_launch_template.app_lt.id
    version = "$Latest"
  }
  health_check_type         = "EC2"
  health_check_grace_period = 300

  tag {
    key                 = "Name"
    value               = "AppASG"
    propagate_at_launch = true
  }
}

# CloudWatch Metric Alarm
resource "aws_cloudwatch_metric_alarm" "cpu_alarm_high" {
  alarm_name          = "cpu-utilization-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "This metric monitors high CPU utilization"
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app_asg.name
  }

  alarm_actions = [aws_autoscaling_policy.scale_up.arn]
}

# Scaling Policy - Scale Up
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "scale-up"
  scaling_adjustment     = 1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.app_asg.name
}
```
