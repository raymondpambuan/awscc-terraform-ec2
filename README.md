# 🖥️ IaC: Terraform with EC2 ⌨️

**Terraform** is an open-source Infrastructure as Code (IaC) tool developed by HashiCorp. It enables users to define, provision, and manage infrastructure using a declarative configuration language (HCL). Terraform is widely used for cloud automation, DevOps workflows, and infrastructure scaling. 🚀

**Disclaimer:** This activity uses Windows 10/11 OS.

## 📋 Requirements
1. [VS Code](https://code.visualstudio.com/download)
   - (Optional) Extensions: HashiCorp Terraform, Prettier
3. [Git Bash](https://git-scm.com/downloads) or [SSH for PowerShell](https://www.ionos.com/digitalguide/server/configuration/powershell-ssh/)
   - *Note: Git Bash has a built-in SSH, while PowerShell does not.*
4. [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
   - Verify installation using `terraform -help` in any terminal.
5. AWS Account (Preferrably an admin [IAM Account](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html))
6. [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
   - Verify with `aws --version` in Command Prompt
   
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

provider "aws" { 
   region     = "ap-southeast-1"   
}
</pre>

3. Save and run the command `terraform init` using the terminal to initialize Terraform in the current working directory.
4. Log in to your AWS console and navigate to `IAM`.

![image](https://github.com/user-attachments/assets/f5c34477-3196-4d2f-a51d-7d1b8bb434bf)

6. Create an IAM User named `terraform-admin`.
7. Attach the policy `AdministratorAccess` directly, or add user to a group if you have an existing group with `AdministratorAccess`. Then, click `Next` and `Create User`.
8. Select the created user and navigate to `Security Credentials` tab. Under `Access Keys`, click `Create Access Key`.
9. Select `Command Line Interface` for use case, check the confirmation box, and click `Next`.
10. Name it `terraform-key`, then click `Create`.
11. Copy both keys and store them in a notepad. You can also download the `.csv`.
12. For secure local credential set up, enter `aws configure` in a terminal and fill up the following sequentially:
    - `[YOUR ACCESS KEY]`
    - `[YOUR SECRET KEY]`
    - `ap-southeast-1`
    - `json`

## Part 2: Writing Terraform Code for EC2 Provisioning
1. In `main.tf`, under the `provider` block, remove the region parameter since it is already configured in your AWS CLI. With the same reason and for security purposes, the access keys will not be added here.

<pre>
provider "aws" { }
</pre>

2. Copy the [RSA resource](https://registry.terraform.io/providers/hashicorp/tls/latest/docs/resources/private_key) block and paste it to `main.tf`. This is to set up the keys for SSH connection.

<pre>
resource "tls_private_key" "rsa_4096" {
  algorithm = "RSA"
  rsa_bits  = 4096
}
</pre>

3. Create a new file named `variables.tf` to isolate the variables and follow the recommended code practice.
4. Create a variable block for the `key_name` of the PEM file for SSH connection key authentication.

<pre>
variable "key_name" {
  type        = string
  description = "Name of the key pair"
}
</pre>

5. Create a new file named `terraform.tfvars` to set the actual values of the variables. Add a line for `key_name` and set it as `"instance_pem"`.
6. Copy the [AWS key pair](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/key_pair) block and paste it to `main.tf`. A variable reference `var.` is now used for modularity.

<pre>
resource "aws_key_pair" "key_pair" {
  key_name   = var.key_name
  public_key = tls_private_key.rsa_4096.public_key_openssh
}
</pre>

7. Create a local file for the private key. This allows SSH access in your local machine as it serves as authentication keys.

<pre>
resource "local_file" "private_key" {
  content = tls_private_key.rsa_4096.private_key_openssh
  filename = var.key_name
}
</pre>

8. In `variables.tf`, add variables for EC2 provisioning, namely `ami` for the Amazon Machine Image to be used for the instance, `instance_type` for the type of instance to be launched, and `instance_name` for the name of the instance.

<pre>
variable "ami" {
  type        = string
  description = "Value of the AMI to use for the instance"
}

variable "instance_type" {
  type        = string
  description = "Type of instance to launch"
}

variable "instance_name" {
  type        = string
  description = "Name of the instance"
}
</pre>

9. In `terraform.tfvars`, set the new variables' values. For `ami`, we will use the latest version of Ubuntu, with value `"ami-01938df366ac2d954"`. For `instance_type`, `"t2.micro"` will be used. For `instance_name`, use `"public_instance"`.
10. Now, in `main.tf`, create an [AWS instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance) block for the EC2 instance.

<pre>
resource "aws_instance" "public_instance" {
  ami                    = var.ami
  instance_type          = var.instance_type
  key_name               = aws_key_pair.key_pair.key_name
   
  tags = {
    Name = var.instance_name
  }
}
</pre>

11. To access the instance with SSH, a [security group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) block must be created. Add the following blocks in `main.tf` to allow SSH connection.

<pre>
resource "aws_security_group" "ssh_access" {
  name = "ssh_access"
}

resource "aws_vpc_security_group_ingress_rule" "ssh" {
  security_group_id = aws_security_group.ssh_access.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
}
</pre>

12. The security group must be attached to the EC2 instance. Hence, under the resource block `"aws_instance" "public_instance"`, add an argument `vpc_security_group_ids` with value `[aws_security_group.ssh_access.id]`.
13. To immediately obtain the public IPv4 address of the EC2 instance, we can set it in `outputs.tf`. Create a new file named as `outputs.tf` and add the corresponding block.

<pre>
output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.public_instance.public_ip
}
</pre>

## Part 3: Running Terraform and Verifying EC2 with SSH
1. Save and run `terraform init` in the terminal.
2. Run `terraforrm plan` and observe the output. It should display the infrastructure resources to be created, changed, or destroyed.
3. Run `terraform validate` to check for validity.
4. Run `terraform apply` to run the configured EC2 instance and other AWS resources. If prompted for approval, enter `yes` and wait.
5. To verify if the EC2 instance is launched, go to your AWS console, navigate to `EC2`. It should be shown under the `Instances` dashboard, refresh if needed. **Screenshot the page beside the terminal with terraform apply output**.

![image](https://github.com/user-attachments/assets/79d0272a-c9d5-4b70-a3c7-2c5b97eaa205)

6. Obtain the file path of the `PEM` file.
   - Using Git Bash in VS Code: `readlink -f [filename]`
   - Using VS Code GUI: right-click the file in the `Explorer` tab and select `Copy Path`.
     ![image](https://github.com/user-attachments/assets/52bfea85-fe77-40cb-a316-d14faaa7c274)

7. (For Windows if sudo disabled) Go to Settings > System > For developers > set Enable sudo On. You can disable this after accomplishing the activity.

![image](https://github.com/user-attachments/assets/c0c700bb-8f1e-437b-a667-6904be32f635)

8. Initiate an SSH connection into the instance. 
   - Using Git Bash:`sudo ssh -i [FILE_PATH_OF_PEM] ubuntu@[Public_IPv4_EC2Instance]`
    Example: `sudo ssh -i /c/Users/Raymond/Projects/iac-terraform/ec2/mon_pem ubuntu@234.567.891.011`
   - Using PowerShell: `ssh -i [FILE_PATH_OF_PEM] ubuntu@[Public_IPv4_EC2Instance]`
    Example: `ssh -i C:\Users\Raymond\Projects\iac-terraform\ec2\mon_pem ubuntu@234.567.891.011`
   - *Note: "user@" depends on AMI selected (e.g. ec2-user default for Amazon Linux 2023)*
9. (For Devices with Multiple Users) You might encounter errors regarding the PEM file. Check this [article](https://superuser.com/questions/1296024/windows-ssh-permissions-for-private-key-are-too-open) first.
10. To verify if you are already in an SSH session, the CLI should be similar to `user@ip-123-45-67-891:~$`
11. Install `net-tools` by entering `sudo apt install net-tools`. Notice that the install will timeout. This is due to a lacking security group rule for outbound connections. Enter `logout` to the SSH session CLI.
12. Go back to VS Code, in `main.tf`, add a security group egress rule block to allow any access for outbound connections from the EC2 instance. (Optional: Set `"0.0.0.0/0"` as a Terraform variable)

<pre>
resource "aws_vpc_security_group_egress_rule" "allow_all_traffic_ipv4" {
  security_group_id = aws_security_group.ssh_access.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1" # semantically equivalent to all ports
}
</pre>

12. Since there were updates to the infrastructure, we need to perform `terraform apply` again. Save and enter `terraform plan`, `terraform validate`, and `terraform apply` sequentially. Once the updates are applied, initiate an SSH session again using the commands from Step 8.
13. Attempt to install `net-tools` again by entering `sudo apt install net-tools`.
14. After a successful install, enter `ifconfig` to get the private IPv4 address.
15. Verify if the private IPv4 address is the same with the one displayed in the AWS EC2 instance dashboard. Take a **screenshot** with the terminal and AWS dashboard side-by-side. Log out of the SSH session once done.

![image](https://github.com/user-attachments/assets/bf114d35-d09a-46bc-a5e4-f0189c956828)

16. To clean up the resources provisioned to AWS, go back to VS code and enter `terraform destroy` to the terminal. Enter `yes` after the corresponding prompt.
17. Go back to AWS dashboard and ensure the `Instance State` is  displayed as `Terminated`. If the instance remains displayed as `Terminated` after several hours, check this [article](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesShuttingDown.html#terminated-instance-still-displaying).
