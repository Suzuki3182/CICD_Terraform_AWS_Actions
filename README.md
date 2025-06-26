Certainly! Below is a comprehensive `README.md` file for the CI/CD pipeline setup using Terraform and GitHub Actions.

---

# CI/CD Pipeline with Terraform and GitHub Actions

This repository contains the necessary scripts and configurations to set up a CI/CD pipeline for deploying applications to an AWS EC2 instance using Terraform and GitHub Actions.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Terraform Scripts](#terraform-scripts)
- [GitHub Actions Workflow](#github-actions-workflow)
- [Configuration Management](#configuration-management)
- [Security](#security)
- [Deployment](#deployment)
- [Maintenance](#maintenance)
- [Contributing](#contributing)

## Prerequisites

Before you begin, ensure you have the following prerequisites:

1. **AWS Account**: You need an AWS account to provision the infrastructure.
2. **Terraform Installed**: Ensure Terraform is installed on your local machine.
3. **GitHub Account**: A GitHub account to host the repository and manage secrets.
4. **SSH Key Pair**: An SSH key pair for secure access to the EC2 instance.

## Terraform Scripts

The `main.tf` file contains the Terraform configuration to provision the AWS infrastructure. It includes:

- **VPC**: A Virtual Private Cloud (VPC) to host the EC2 instance.
- **Subnets**: Public subnets for the EC2 instance.
- **Internet Gateway**: To allow internet access for the EC2 instance.
- **Route Table**: Routing rules for the public subnet.
- **Security Group**: Rules to allow SSH and HTTP access.
- **IAM Role and Policy**: For managing permissions on the EC2 instance.
- **EC2 Instance**: The instance where the application will be deployed.

### main.tf
```hcl
# Provider configuration
provider "aws" {
  region = "us-west-2"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "main-vpc"
  }
}

# Subnets
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  tags = {
    Name = "public-subnet"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "main-igw"
  }
}

# Route Table for Public Subnet
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
  tags = {
    Name = "public-route-table"
  }
}

# Associate Route Table with Public Subnet
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group for EC2 Instance
resource "aws_security_group" "instance" {
  name        = "instance-sg"
  description = "Allow SSH and HTTP access"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
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
    Name = "instance-sg"
  }
}

# IAM Role for EC2 Instance
resource "aws_iam_role" "instance" {
  name = "ec2-instance-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com"
        },
      },
    ],
  })
}

# IAM Policy for EC2 Instance
resource "aws_iam_policy" "instance" {
  name        = "ec2-instance-policy"
  description = "Policy for EC2 instance"

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ],
        Effect = "Allow",
        Resource = [
          "arn:aws:s3:::my-bucket",
          "arn:aws:s3:::my-bucket/*"
        ],
      },
    ],
  })
}

# Attach IAM Policy to IAM Role
resource "aws_iam_role_policy_attachment" "instance" {
  role       = aws_iam_role.instance.name
  policy_arn = aws_iam_policy.instance.arn
}

# EC2 Instance
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0" # Replace with your desired AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id
  security_groups = [aws_security_group.instance.name]
  iam_instance_profile = aws_iam_role.instance.name

  user_data = <<-EOF
              #!/bin/bash
              sudo apt-get update -y
              sudo apt-get install -y apache2
              sudo systemctl start apache2
              sudo systemctl enable apache2
              EOF

  tags = {
    Name = "web-instance"
  }
}
```

## GitHub Actions Workflow

The `.github/workflows/ci-cd.yml` file defines the CI/CD pipeline using GitHub Actions. It includes steps to:

- Checkout the code from the repository.
- Set up Node.js environment.
- Install dependencies.
- Run tests.
- Build the application.
- Deploy the application to the EC2 instance using SSH.

### .github/workflows/ci-cd.yml
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Set up Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '14'

    - name: Install dependencies
      run: npm install

    - name: Run tests
      run: npm test

    - name: Build application
      run: npm run build

    - name: Deploy to EC2
      env:
        EC2_HOST: ${{ secrets.EC2_HOST }}
        EC2_USER: ${{ secrets.EC2_USER }}
        SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      run: |
        mkdir -p ~/.ssh
        echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa
        scp -o StrictHostKeyChecking=no -r ./dist $EC2_USER@$EC2_HOST:/var/www/html

    - name: Restart Apache
      env:
        EC2_HOST: ${{ secrets.EC2_HOST }}
        EC2_USER: ${{ secrets.EC2_USER }}
        SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      run: |
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa $EC2_USER@$EC2_HOST "sudo systemctl restart apache2"
```

## Configuration Management

The EC2 instance is configured using a user data script in the Terraform configuration. This script installs Apache and starts the service.

### User Data Script
```bash
#!/bin/bash
sudo apt-get update -y
sudo apt-get install -y apache2
sudo systemctl start apache2
sudo systemctl enable apache2
```

## Security

- **GitHub Secrets**: The pipeline uses GitHub Secrets to handle sensitive information such as the EC2 host, user, and SSH private key.
- **Security Group**: The EC2 instance has a security group that allows only necessary ports (SSH and HTTP).

## Deployment

To deploy the infrastructure and application, follow these steps:

1. **Initialize Terraform**:
   ```bash
   terraform init
   ```

2. **Apply Terraform Configuration**:
   ```bash
   terraform apply
   ```

3. **Set Up GitHub Secrets**:
   - Go to your GitHub repository settings.
   - Navigate to `Settings > Secrets > Actions`.
   - Add the following secrets:
     - `EC2_HOST`: The public IP or DNS of the EC2 instance.
     - `EC2_USER`: The username for the EC2 instance (e.g., `ubuntu`).
     - `SSH_PRIVATE_KEY`: The private key for SSH access to the EC2 instance.

4. **Push Code to GitHub**:
   - Commit and push your code to the `main` branch.
   - The GitHub Actions workflow will automatically trigger the CI/CD pipeline.

## Maintenance

- **Terraform State Management**: Ensure you manage the Terraform state file securely.
- **Regular Updates**: Regularly update dependencies and Terraform scripts to ensure compatibility and security.
- **Monitoring**: Set up monitoring and alerting for the EC2 instance to ensure high availability.

## Contributing

Contributions are welcome! Please follow these guidelines:

1. Fork the repository.
2. Create a new branch for your feature or bug fix.
3. Make your changes and ensure they follow the coding standards.
4. Submit a pull request with a clear description of your changes.

---

This README provides a comprehensive guide to setting up and maintaining a CI/CD pipeline using Terraform and GitHub Actions.
