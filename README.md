# 🖥️ IaC: Terraform with EC2 ⌨️

Terraform Lorem Ipsum

**Disclaimer:** This activity uses Windows 10/11 OS.

## 📋 Requirements
1. [VS Code](https://code.visualstudio.com/download)
2. [Git Bash](https://git-scm.com/downloads) or [SSH for PowerShell](https://www.ionos.com/digitalguide/server/configuration/powershell-ssh/)
   - *Note: Git Bash has a built-in SSH, while PowerShell does not.*
3. [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
   - Verify installation using `terraform -help` in any terminal.
4. AWS Account (Preferrably an admin IAM Account)
   
## 🎯 Objectives
1. Setting Up Environment and AWS Credentials
2. Writing Terraform Code for EC2 Provisioning
3. Running Terraform and Verifying EC2 with SSH

## Part 1: Setting Up Environment and AWS Credentials
1. Create a folder named `awscc-iac-terraform` in your preferred directory and open it with VS Code.
2. Create a file `main.tf` and copy the [provider code](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) below as its initial content.

<pre>
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
  region = "us-east-1"
}
</pre>

3. Log in to your AWS console and navigate to `IAM`.
4. Create an IAM User named `terraform-admin`.
5. Attach the policy `AdministratorAccess`, then click `Create user`.
6. Select the created user and navigate to `Security Credentials` tab. Under `Access Keys`, click `Create Access Key`.
7. Select `Command Line Interface` for key type, check the confirmation box, and click `Next`.
8. Name it `terraform-key`, then click `Create`.
9. Copy both keys and store them in a notepad.

## Part 2: Writing Terraform Code for EC2 Provisioning
