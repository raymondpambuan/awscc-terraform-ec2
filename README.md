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
  region = "ap-southeast-1"
}
</pre>

3. Run the command `terraform init` using the terminal.
4. Log in to your AWS console and navigate to `IAM`.
5. Create an IAM User named `terraform-admin`.
6. Attach the policy `AdministratorAccess`, then click `Create user`.
7. Select the created user and navigate to `Security Credentials` tab. Under `Access Keys`, click `Create Access Key`.
8. Select `Command Line Interface` for key type, check the confirmation box, and click `Next`.
9. Name it `terraform-key`, then click `Create`.
10. Copy both keys and store them in a notepad.

## Part 2: Writing Terraform Code for EC2 Provisioning
1. In `main.tf`, add `access_key` and `secret_key` parameters under the `provider` block with your corresponding keys.

<pre>
provider "aws" {
  region     = "ap-southeast-1"
  access_key = "[YOUR ACCESS KEY]"
  secret_key = "[YOUR SECRET KEY]"
}
</pre>

2. Copy the [RSA resource](https://registry.terraform.io/providers/hashicorp/tls/latest/docs/resources/private_key) block and paste it to `main.tf`. Rename the object to `rsa_4096`.

<pre>
resource "tls_private_key" "rsa_4096" {
  algorithm = "RSA"
  rsa_bits  = 4096
}
</pre>

3. Create a variable block: `variable "key_name" {}`.
4. Copy the [AWS key pair](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/key_pair) block and paste it to `main.tf`. Change `deployer` to `key_pair`, `key_name` value to `var.key_name` and `public_key` value to `tls_private_key.rsa_4096.public_key_openssh`

<pre>
resource "aws_key_pair" "key_pair" {
  key_name   = var.key_name
  public_key = tls_private_key.rsa_4096.public_key_openssh
}
</pre>

5. Create a local file for the private key.

<pre>
resource "local_file" "private_key" {
  content = tls_private_key.rsa_4096.private_key_openssh
  filename = var.key_name
}
</pre>

6. Create an [AWS instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance) block for the EC2 instance. Rename the object name to `public_instance`.
commment: consider ami lookup^^
<pre>
resource "aws_instance" "public_instance" {
  ami           = "ami-0672fd5b9210aa093"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.key_pair.key_name

  tags = {
    Name = "public_instance"
  }
}
</pre>

## Part 3: Running Terraform and Verifying EC2 with SSH
1. Run `terraform init` in the terminal.
2. Run `terraforrm plan` and enter a value for `key_name` using the format `[nickname]_pem`
3. Run `terraform validate` to check for validity.
4. Run `terraform apply` to run the configured EC2 instance, and if prompted, enter the value set for `key_name`. If prompted again, enter `yes` and wait. **Screenshot the output**.
5. To verify if the EC2 instance is launched, go to your AWS console, navigate to `EC2`. It should be shown under the `Instances` dashboard, refresh if needed. **Screenshot the page**.
6. Click `Instance` and go to `Security` > `Security Groups`. Check the existing rule. Notice there is not rule for SSH connection.
7. Go back to `main.tf` in VS Code. Add a block for rule type SSH and source to 0.0.0.0/0.
8. Obtain the file path of the `PEM` file.
   - Using Git Bash in VS Code: `readlink -f [filename]`
   - Using VS Code GUI: right-click the file in the `Explorer` tab and select `Copy Path`.
9. (For Windows if sudo disabled) Go to Settings > System > For developers > set Enable sudo On.
10. Initiate an SSH connection into the instance.
   - Using Git Bash:`sudo ssh -i [FILE_PATH_OF_PEM] ubuntu@[Public_IPv4_DNS_EC2Instance]`
    Example: `sudo ssh -i /c/Users/Raymond/Projects/iac-terraform/ec2/mon_pem ubuntu@ec2-234-567-891-011.ap-southeast-1.compute.amazonaws.com`
   - Using PowerShell: `ssh -i [FILE_PATH_OF_PEM] ubuntu@[Public_IPv4_DNS_EC2Instance]`
   Example: `ssh -i C:\Users\Raymond\Projects\iac-terraform\ec2\mon_pem ubuntu@ec2-122-248-228-101.ap-southeast-1.compute.amazonaws.com`
11. To verify if you are already in an SSH session, the CLI should be similar to `ubuntu@ip-123-45-67-891:~$`
12. Install net-tools by entering `sudo apt install net-tools`.
13. After a successful install, enter `ifconfig` to get the private ip address.
14. Check if the private IPv4 address is the same with the one displayed in the AWS EC2 instance dashboard. Take a **screenshot** with the terminal and AWS dashboard side-by-side.
15. Go back to VS code, enter `terraform destroy` to the terminal. Enter the `PEM` filename earlier, then enter `yes` after the corresponding prompts.
16. Go back to AWS dashboard and ensure the `Instance State` is  displayed as `Terminated`. If the instance remains displayed as `Terminated` after several hours, check this [article](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesShuttingDown.html#terminated-instance-still-displaying).
