
# Multi-Tier Web Application Infrastructure Using Terraform

## Create a Ec2 instance
Create a Ec2 instance config (Amazon linux 2023) t2.micro. Install httpd server and mysql client. Once created stop this ec2 instance and generate an Ami out on it.

## Create provider.tf file
```bash
provider "aws" {
  region = "us-east-1"
}
