---
title: "AWS & Terraform Part I"
date: 2022-07-31T18:46:56-04:00
draft: false
---

# AWS & Terraform
-------
Programmable infrastructure allow professionals to manage on-premises and cloud resources through code instead of  management platforms and manual methods traditionally used by IT teams such as the AWC CLI or AWS Console. 

Infastructure As Code (IaC) is an IT practice of codifying and mananging IT infastructure as software/code. Terraform interacts with cloud and on-prem Application Programming Interfaces (APIs) to configure infastructure such as EC2 instances, Virtual Private Cloud (VPC) and Security Groups.  IaC allows for version control and fine tuned configuration management for any organization's IT  posture. Terraform works through a series of configuration files `.tf`  written in HashiCorp Configuration Lanuage (HCL). These `.tf` files work in concert to define the resources required and allow for seamless changes based on need. 

This demo will create the following resources in AWS for a "development" environment and a "production" environment. 

- VPC
- Public & Private Subnet
-  NAT Gateway
- Internet Gateway
- Key-Pair (SSH Key)
- SSH Security Group (needed to ssh into the demo instance)
- AWS EC2 Instance

## Prerequisites: 

- Preferred Linux OS (I am using Manjaro for this demo)
- AWS Account & Command Line Interface w/ access key and secrets
	- You may use hard coded AWS credentials in your `.aws` directory or utilize `aws-vault` to store access keys and secrets securely. 
- Git
- Basic knowledge of Docker and web apps
-(Optional) Text Editor or IDE such as VSCode

## Terraform Environment: 

Install terraform via  your preferred package manager:

```bash
# Debian/Ubuntu:
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl

curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

sudo apt-get update && sudo apt-get install terraform

#Arch/ Manjaro:
sudo pacman -S terraform

```


Once installed verify install:
```bash
terraform -help

```

Clone the demo GitHub repository and cd into the directory:

```bash 
git clone https://github.com/alexrf45/aws-terraform-practice.git

cd aws-terraform-practice
```

Below is a breakdown of the provided files in the directory: 

```
.
├── demo
│   ├── docker.sh
│   ├── ec2_instance.tf
│   ├── ec2_security_group.tf
│   ├── outputs.tf
│   ├── provider.tf
│   ├── vars.tf
│   └── vpc_creation.tf
├── dev.auto.tfvars
├── prod.auto.tfvars
├── provider.tf
└── README.md

```

## *docker.sh* - A simple bash script that installs docker and enables non-root permissions. Setting up scripts for use in config files can help provide a consistent environement for developers and administrators. 

```bash
#! /bin/bash

sudo apt-get update
sudo apt-get upgrade
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker ubuntu


```

## ec2_instance.tf - The config file for creating the demo EC2 Instance. 

```hcl

resource "aws_key_pair" "demo_ssh_key" {
  key_name   = var.key_name
  public_key = file("~/.ssh/demo-key.pub")
}

resource "aws_instance" "terraform_ec2" {
  ami                         = var.ami
  instance_type               = var.instance_type
  associate_public_ip_address = "true"
  key_name                    = var.key_name
  vpc_security_group_ids      = [aws_security_group.terraform-ssh.id]
  subnet_id                   = aws_subnet.dev-public-subnet.id
  user_data                   = file("docker.sh")
  tags                        = var.resource_tags
}

```



## ec2_security_group.tf - This config files creates a simple security group for enabling SSH access to our demo EC2 instance. 

```hcl 
resource "aws_security_group" "terraform-ssh" {
  name        = "terraform-ssh-sg"
  description = "Allow SSH inbound traffic"
  vpc_id      = aws_vpc.dev.id

  ingress {
    description = var.description
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = var.resource_tags
}

```

## outputs.tf - This config file exports outputs from the creation of each resource for use in other resource config files.  (This is neccessary to add the security group to the EC2 instance and configure the VPC properly)

```hcl 
output "vpc_id" {
  description = "The ID of the VPC"
  value       = try(aws_vpc.dev.id, "")
}

output "vpc_cidr_block" {
  description = "The CIDR block of the VPC"
  value       = try(aws_vpc.dev.cidr_block, "")
}
output "vpc_arn" {
  description = "The ARN of the VPC"
  value       = try(aws_vpc.dev.arn, "")
}

output "public_subnets" {
  description = "List of IDs of public subnets"
  value       = aws_subnet.dev-public-subnet.id
}

output "private_subnets" {
  description = "List of IDs of private subnets"
  value       = aws_subnet.dev-private-subnet.id
}

output "public_ip" {
  description = "The public IP address assigned to the instance, if applicable. NOTE: If you are using an aws_eip with your instance, you should refer to the EIP's address directly and not use `public_ip` as this field will change after the EIP is attached"
  value       = aws_instance.terraform_ec2.public_ip
}

output "id" {
  description = "The ID of the instance"
  value       = aws_instance.terraform_ec2.id
}

```

## provider.tf - This config files sets the provider as AWS to ensure we are interacting with the AWS API. You can add other providers as needed to keep an easy to reference list for your config files. 

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }

  required_version = ">= 1.2.0"
}

provider "aws" {
  region = var.aws_region
}

```

## vars.tf - This config files declares variables needed for each resource config file. This config file also allows us to utilize a `.tfvars` file to pass variable values to the resource configuration during run time. A variables file helps reduce the need for manual alteration of config files. 

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}
variable "resource_tags" {
  description = "Tags to set for all resources"
  type        = map(string)
  default = {
    project     = "test-project",
    environment = "dev"
    Name        = "Development"
  }
}
variable "dev_vpc_cidr" {}

variable "public_subnets" {}

variable "private_subnets" {}

variable "ami" {
  type    = string
  default = "ami-052efd3df9dad4825"

}

variable "instance_type" {
  description = "EC2 Instance type"
  type        = string
  default     = "t2-micro"
}

variable "key_name" {
  description = "Name of the associated ec2 key pair"
  type        = string
  default     = "dev-demo"
}

variable "description" {
  description = "Description of security group"
  type        = string
  default     = "SSH"
}
```

## vpc_creation.tf - This config file creates a new VPC network for our demo. 

```hcl
resource "aws_vpc" "dev" {
  cidr_block           = var.dev_vpc_cidr
  instance_tenancy     = "default"
  enable_dns_support   = "true"
  enable_dns_hostnames = "true"
  tags                 = var.resource_tags
}

resource "aws_internet_gateway" "dev-IGW" { # Creating Internet Gateway
  vpc_id = aws_vpc.dev.id                   # vpc_id will be generated after we create VPC
}

resource "aws_subnet" "dev-private-subnet" {
  vpc_id     = aws_vpc.dev.id
  cidr_block = var.private_subnets # CIDR block of private subnets
  tags       = var.resource_tags
}

resource "aws_subnet" "dev-public-subnet" {
  vpc_id     = aws_vpc.dev.id
  cidr_block = var.public_subnets # CIDR block of public subnets
  tags       = var.resource_tags
}
# Route table for Public Subnet's
resource "aws_route_table" "dev-public-rt" { # Creating RT for Public Subnet
  vpc_id = aws_vpc.dev.id
  route {
    cidr_block = "0.0.0.0/0" # Traffic from Public Subnet reaches Internet via Internet Gateway
    gateway_id = aws_internet_gateway.dev-IGW.id
  }
  tags = var.resource_tags
}
#Route table for Private Subnet's
resource "aws_route_table" "dev-private-rt" { # Creating RT for Private Subnet
  vpc_id = aws_vpc.dev.id
  route {
    cidr_block     = "0.0.0.0/0" # Traffic from Private Subnet reaches Internet via NAT Gateway
    nat_gateway_id = aws_nat_gateway.NATgw.id
  }
  tags = var.resource_tags
}

#Route table Association with Public Subnet's
resource "aws_route_table_association" "dev-public-rt-association" {
  subnet_id      = aws_subnet.dev-public-subnet.id
  route_table_id = aws_route_table.dev-public-rt.id
}
#Route table Association with Private Subnet's
resource "aws_route_table_association" "dev-private-rt-association" {
  subnet_id      = aws_subnet.dev-private-subnet.id
  route_table_id = aws_route_table.dev-private-rt.id
}
resource "aws_eip" "devnatIP" {
  vpc  = true
  tags = var.resource_tags
}
#Creating the NAT Gateway using subnet_id and allocation_id
resource "aws_nat_gateway" "NATgw" {
  allocation_id = aws_eip.devnatIP.id
  subnet_id     = aws_subnet.dev-public-subnet.id
  tags          = var.resource_tags
}

```

## dev.auto.tfvars - `.tfvars` files allow us to manage variable assignments systematically in a file with the extension .tfvars or .tfvars.json. 

```hcl
aws_region = "us-east-1"

availability_zone_names = [
  "us-east-1d",
  "us-east-1e",
  "us-east-1f"
]

instance_type = "t2.micro"

key_name = "dev-demo"


ami = "ami-052efd3df9dad4825"

dev_vpc_cidr    = "10.0.0.0/16"
public_subnets  = "10.0.1.0/24"
private_subnets = "10.0.2.0/24"

```

## prod.auto.tfvars - Same as `dev.auto.tfvars` with a few changes to simulate a production environment. We will not use all the variable declared here today. 

```hcl
aws_region = "us-east-1"

availability_zone_names = [
  "us-east-1a",
  "us-east-1b",
  "us-east-1c"
]

network = "10.0.1"

instance_type = "t2.micro"

key_name = "demo-prod"

volume_tags = {
  "Name"  = "Demo"
  "Owner" = "Demo-Prod"
}

ami = "ami-052efd3df9dad4825"

dev_vpc_cidr    = "10.0.0.0/16"
public_subnets  = "10.0.1.0/24"
private_subnets = "10.0.2.0/24"

```



A `providers.tf` file is maintained in the root directory and can be copied to new resource directories as needed i.e `demo/`




# Each file provides the neccessary declarations to create AWS resources. Let's prepare our enviornment for running terraform commands: 

## Terraform Initialize: 

In order for terraform to interact with the AWS API, lets set up the `demo/` directory for use in terraform: 

```

$ cd demo 

$ terraform init

Initializing the backend...
   
Initializing provider plugins...                                          
- Reusing previous version of hashicorp/aws from the dependency lock file
- Using previously-installed hashicorp/aws v4.22.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.
    
If you ever set or change modules or backend configuration for Terraform,                                                                            
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.

```

- I have previously ran the command in the `demo`  directory so it reads from the .terraform.lock.hcl file which contains the provider (AWS) info stated in the `provider.tf` file. 
--------

## SSH Access: 
In order to access our EC2 instance we need to create a SSH key on our local machine. The  `ec2_instance.tf`  file copies our public key associated with the key for use in creating an AWS Key-pair resource. Skip the prompts to add a passphrase as this key will only be used for the duration of the demo. 

```bash
ssh-keygen -t ed25519 -C "terraform demo" -f ~/.ssh/demo-key #creates an ssh key in your ssh directory. If you don't have one, create it. 

```

Verify the key and it's associated public key are on your local machine, you should see two files named demo-key and demo-key.pub: 

```
ls -la ~/.ssh/

```

Open the  `ec2_instance.tf` file and verify the aws_key_pair resource reflects the file path of public key. If you created a key with a different name, change it here to ensure it copies the correct public key: 

```hcl
resource "aws_key_pair" "demo_ssh_key" {
  key_name   = var.key_name
  public_key = file("~/.ssh/demo-key.pub")
}
```

Now we will be able to access the EC2 instance once it is created. 

--------

## Terraform Validate: 

- Once initialized, type `terraform validate` to verify the configuration is valid.
```
Success! The configuration is valid.
```
------

## Terraform Plan:

The `terraform validate` command ensures the configuration files work together and declare resources appropriately. `terraform plan`  creates a plan file that is used as the final instructions to create the resources and validates the configuration is correct. 

In this demo we are utilizing a `.tfvars` file to introduce inputs for variables at the plan phase of deployment (`dev.auto.tfvars`) . The `.tfvars` file by default is named `terraform.tfvars` but that can get confusing with thousands of resources or locally built modules (our demo/ directory is an example of a module that could be referenced in another config file in the root module) Terraform also recognizes `.auto.tfvars` . 

The `terraform plan` command has two useful arguments neccessary for the demo: 

`-var-file="../dev.auto.tfvars"`  - This references the inputs for variables declared in our demo directory/module and passes them along to the configuration as needed. 

`-out=demo-plan`  - This outputs a plan file with the specified name. This is usual for testing or setting up different plan files based on use case. For the demo, name it something simple you can remember. 

Let's run the `terraform plan` command: 

```bash
terraform plan -var-file="../dev.auto.tfvars" -out=demo-plan
```

You should see alot of output showing resources and changes followed by instructions on how to create the resources at the bottom of your terminal: 

![[plan.png]]
You can also review what resources will be created with the `terraform show` command: 

```bash
terraform show "demo-plan"
```

Below is an example output of where the `ec2_instance.tf` file copies the public key for creating the AWS key pair: 

![[keypair.png]]

--------------

## Terraform Apply:

Now that we have created a terraform plan file, we can run `terraform apply` to begin creation of the AWS resources:

```bash

terraform apply "demo-plan"

```

Terraform will begin creating resources and provide updates as it begins and finishes creating resources:

Example output: 
```
aws_vpc.dev: Creating...                                                                                                                                                                               
aws_key_pair.demo_ssh_key: Creating...                                                                                                                                                                 
aws_eip.devnatIP: Creating...
aws_key_pair.demo_ssh_key: Creation complete after 0s [id=dev-demo]
aws_eip.devnatIP: Creation complete after 1s [id=eipalloc-01501eabaec47d25b]
aws_vpc.dev: Still creating... [10s elapsed]
aws_vpc.dev: Creation complete after 12s [id=vpc-0c858786e38f2f210]
aws_subnet.dev-public-subnet: Creating...
aws_internet_gateway.dev-IGW: Creating...
aws_subnet.dev-private-subnet: Creating...
aws_security_group.terraform-ssh: Creating...
aws_internet_gateway.dev-IGW: Creation complete 


```

Resource creation should take about 2-4 minutes depending on proximity from the us-east-1 AWS region. You should see the following type of output in your terminal (**Resource ARNs, IDs and Public IP will be different): 

```
Apply complete! Resources: 13 added, 0 changed, 0 destroyed.

Outputs:

id = "i-066df87335634ad81"
private_subnets = "subnet-0b1eef62196293730"
public_ip = "100.26.243.84" 
public_subnets = "subnet-097d16be921f7a1b3"
vpc_arn = "arn:aws:ec2:us-east-1:710932396715:vpc/vpc-0c858786e38f2f210"
vpc_cidr_block = "10.0.0.0/16"
vpc_id = "vpc-0c858786e38f2f210"


```
--------------

## Verify EC2 SSH Access: 

Now we can SSH into our EC2 instance and verify Docker is installed as referenced in user_data arguement in the `ec2_instance.tf` file:

```bash
ssh -i ~/.ssh/demo-key ubuntu@PUBLICIP

```


--------------

## Testing EC2 functionality using Docker: 

Lets' pull down a simple REST API container and run it:

In your ssh terminal clone this repo down and cd into the directory: 
```bash
git clone https://github.com/eaccmk/node-app-http-docker

cd node-app-http-docker

```

Repo contents: 

```bash

ubuntu@ip-10-0-1-88:~/node-app-http-docker$ tree .
.
├── Dockerfile
├── LICENSE
├── README.md
├── app.js
├── controller.js
├── data.js
├── package.json
└── utils.js

0 directories, 8 files

```

First let's build the docker image (this take about 1-2 minutes): 

```
docker build -t rest-api . 

```

Verify the image was created:

```

docker image ls 

ubuntu@ip-10-0-1-88:~/node-app-http-docker$ docker image ls
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
rest-api     latest    46d5f173ab6d   28 seconds ago   907MB
node         16        c6b745e900c7   5 days ago       907MB

```

Now let's run the container: 

```
docker run --name restapi --rm -d -p 8080:8080 rest-api:latest

```

We can test the REST API works by using the curl command: 

```bash
curl http://localhost:8080


ubuntu@ip-10-0-1-88:~/node-app-http-docker$ curl http://localhost:8080
Welcome, this is your Home page

```

We can also test the 404 response and a health check:

```bash
curl http://localhost:8080/404

ubuntu@ip-10-0-1-88:~/node-app-http-docker$ curl http://localhost:8080/404
{"message":"Route not found"}

curl http://localhost:8080/health

ubuntu@ip-10-0-1-88:~/node-app-http-docker$ curl http://localhost:8080/health
{"uptime":209.060012355,"message":"OK","timestamp":1658095518470}
```

-------------

## Clean Up & Terraform Destroy: 

Let's stop the container and Log out of our EC2 instance: 

```
docker container stop restapi

exit

```

Before we destory our Terraform resources let's review  the created resources:

```
terraform state list
```

`terraform state` displays our resources by their resource name as specified in the .tf files. You should see the following resource IDs below. This is super helpful for defining resources and referencing them.

Example output: 
```
aws_eip.devnatIP
aws_instance.terraform_ec2
aws_internet_gateway.dev-IGW
aws_key_pair.demo_ssh_key
aws_nat_gateway.NATgw
aws_route_table.dev-private-rt
aws_route_table.dev-public-rt
aws_route_table_association.dev-private-rt-association
aws_route_table_association.dev-public-rt-association
aws_security_group.terraform-ssh
aws_subnet.dev-private-subnet
aws_subnet.dev-public-subnet
aws_vpc.dev

```

`terraform destory`  looks at the terraform state and plan to decide on what resources to destory. We willl also reference the `dev.auto.tfvars` file during command execution to ensure resources are destroyed correctly. 

```
terraform destroy -var-file="../dev.auto.tfvars"
```

Terraform will ouput a list of resources and their changes (destroy) and a prompt to confirm deletion. type yes and press enter. 

Example output: 
```
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: 
```

Resource teardown should take about 3-4 minutes. 

Example output: 
```
aws_internet_gateway.dev-IGW: Destruction complete after 3m41s
aws_vpc.dev: Destroying... [id=vpc-0c858786e38f2f210]
aws_vpc.dev: Destruction complete after 0s

Destroy complete! Resources: 13 destroyed.

```
------------------

## Conclusion: 

Today we created AWS resources using Terraform and sprinkled in a litle Docker for fun. Stay tuned as we integrate version control with Terraform to make updating our infastructure as easy as a git push. 