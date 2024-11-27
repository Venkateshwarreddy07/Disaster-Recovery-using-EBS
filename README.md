# Disaster-Recovery-using-EBS
TERRAFORM CODE:

provider "aws" {
  region = "us-east-1"
}

# Create a Key Pair
resource "aws_key_pair" "key" {
  key_name   = "terraform-key"
  public_key = file("~/.ssh/id_rsa.pub") # Replace with your SSH public key path
}

# Security Group for EC2
resource "aws_security_group" "app_sg" {
  name        = "app_security_group"
  description = "Allow HTTP and SSH traffic"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Allow SSH from anywhere
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Allow HTTP from anywhere
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EBS Volume
resource "aws_ebs_volume" "app_volume" {
  availability_zone = "us-east-1a"
  size              = 10 # Size in GiB
  tags = {
    Name = "app_data_volume"
  }
}

# IAM Role for EC2
resource "aws_iam_role" "ec2_role" {
  name = "ec2_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2_instance_profile"
  role = aws_iam_role.ec2_role.name
}

# Launch Template for EC2 Instances
resource "aws_launch_template" "app_template" {
  name = "app-launch-template"
  image_id      = "ami-0c02fb55956c7d316"  # Replace with your AMI ID
  instance_type = "t3.micro"  # Instance type changed to t3.micro
  key_name      = aws_key_pair.key.key_name
  
  iam_instance_profile {
    name = aws_iam_instance_profile.ec2_profile.name
  }
   
  user_data = base64encode(<<EOF
    #!/bin/bash
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello, this is a failover test!</h1>" > /var/www/html/index.html
    mount /dev/xvdf /mnt
EOF
  )
        
  block_device_mappings {
    device_name = "/dev/xvdf"
    ebs {
      volume_size           = 10
      delete_on_termination = false
      volume_type           = "gp2"
    }
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "app-instance"
    }
  }
}

# Auto Scaling Group using Launch Template
resource "aws_autoscaling_group" "app_asg" {
  desired_capacity     = 1
  max_size             = 3
  min_size             = 1
  vpc_zone_identifier  = ["subnet-0d9409ea9003cd699"]  # Reference subnet properly here
  
  launch_template {
    id      = aws_launch_template.app_template.id
    version = "$Latest"  # Reference the latest version of the launch template
  }

  health_check_type    = "EC2"
  health_check_grace_period = 300
  
  tag {
    key                 = "Name"
    value               = "AutoScaling Instance"
    propagate_at_launch = true
  }
}
![image](https://github.com/user-attachments/assets/2546f780-654f-49c5-826e-0721d6f37903)
