# Project Overview

I set up an EC2 instance in the London (eu-west-2) region using Terraform. The instance is configured with the following specifications:

- **Instance Type:** T2.Micro
- **Storage:** 12GB
- **Region:** London (eu-west-2)
- **Access Configuration:**
  - SSH and RDP: Allowed from a single IP address
  - HTTPS: Open to the world

---

# Terraform Configuration

I first created the `providers.tf` file to define the required providers and AWS configuration:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "eu-west-2"
  # access_key = "my-access-key"
  # secret_key = "my-secret-key"
}

Then, I created the main.tf file with the following resources:

#1 I created a VPC
resource "aws_vpc" "london-vpc" {
  cidr_block       = "25.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  instance_tenancy = "default"

  tags = {
    Name = "london-vpc"
    Environment = "devops"
  }
}

#2 I created an Internet Gateway
resource "aws_internet_gateway" "london-igw" {
  vpc_id = aws_vpc.london-vpc.id

  tags = {
    Name = "london-igw"
    Environment = "devops"
  }
}

#3 I created a Public Route Table
resource "aws_route_table" "london-pubRT" {
  vpc_id = aws_vpc.london-vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.london-igw.id
  }

  tags = {
    Name = "london-pubRT"
    Environment = "devops"
  }
}

#4 I created a Public Subnet in eu-west-2a
resource "aws_subnet" "london-pub-subnet-2a" {
  vpc_id     = aws_vpc.london-vpc.id
  cidr_block = "25.0.0.0/24"
  availability_zone = "eu-west-2a"

  tags = {
    Name = "london-pub-subnet-2a"
    Environment = "devops"
  }
}

#5 I associated the Public Subnet with the Route Table
resource "aws_route_table_association" "london-pubRT-association" {
  subnet_id      = aws_subnet.london-pub-subnet-2a.id
  route_table_id = aws_route_table.london-pubRT.id
}

#6 I created a Security Group
resource "aws_security_group" "london-pub-sg" {
  name        = "london-pub-sg"
  description = "Access to SSH and RDP from a single IP address & https from anywhere"
  vpc_id      = aws_vpc.london-vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["198.167.100.20/32"]
  }

  ingress {
    from_port   = 3389
    to_port     = 3389
    protocol    = "tcp"
    cidr_blocks = ["198.167.100.20/32"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "london-pub-sg"
    Environment = "devops"
  }
}

#7 I created a Network Interface with an IP in the Subnet that I created in Step 4
resource "aws_network_interface" "london-eni-2a" {
  subnet_id       = aws_subnet.london-pub-subnet-2a.id
  private_ips     = ["25.0.0.4"]
  security_groups = [aws_security_group.london-pub-sg.id]
}

#8 I assigned an Elastic IP to the Network Interface created in Step 7
resource "aws_eip" "london-eip1" {
  domain                    = "vpc"
  network_interface         = aws_network_interface.london-eni-2a.id
  associate_with_private_ip = "25.0.0.4"

  depends_on = [aws_instance.london-ec2]
}

#9 I created a Keypair
resource "aws_key_pair" "London-KP" {
  key_name   = "London-KP"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD3F6tyPEFEzV0LX3X8BsXdMsQz1x2cEikKDEY0aIj41qgxMCP/iteneqXSIFZBp5vizPvaoIR3Um9xK7PGoW8giupGn+EPuxIA4cDM4vzOqOkiMPhz5XK0whEjkVzTo4+S0puvDZuwIsdiW9mxhJc7tgBNL0cYlWSYVkz4G/fslNfRPW5mYAM49f4fhtxPb5ok4Q2Lg9dPKVHO/Bgeu5woMc7RY0p1ej6D4CKFE6lymSDJpW0YHX/wqE9+cfEauh7xZcG0q9t2ta6F6fmX0agvpFyZo8aFbXeUBr7osSCJNgvavWbM/06niWrOvYX2xwWdhXmXSrbX8ZbabVohBK41 clydemax4@gmail.com"
}

#10 I launched an EC2 Instance
resource "aws_instance" "london-ec2" {
  ami           = "ami-04ba8620fc44e2264" # eu-west-2
  instance_type = "t2.micro"
  key_name = "London-KP"

  network_interface {
    network_interface_id = aws_network_interface.london-eni-2a.id
    device_index         = 0
  }

  root_block_device {
    volume_size = 12
  }

  tags = {
    Name = "london-ec2"
    Environment = "devops"
  }
}
```
