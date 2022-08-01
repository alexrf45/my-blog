---
title: "AWS & Terraform Part II"
date: 2022-07-31T18:46:56-04:00
draft: false
---

Programmable infrastructure allow professionals to manage on-premises and cloud resources through code instead of  management platforms and manual methods traditionally used by IT teams such as the AWC CLI or AWS Console. 

Infastructure As Code (IaC) is an IT practice of codifying and mananging IT infastructure as software/code. Terraform interacts with cloud and on-prem Application Programming Interfaces (APIs) to configure infastructure such as EC2 instances, Virtual Private Cloud (VPC) and Security Groups.  IaC allows for version control and fine tuned configuration management for any organization's IT  posture. Terraform works through a series of configuration files `.tf`  written in HashiCorp Configuration Lanuage (HCL). These `.tf` files work in concert to define the resources required and allow for seamless changes based on need. 

Part II of this demo will create the following resources in AWS for a "development" environment using Terraform through Github Actions:

- VPC
- Public & Private Subnet
-  NAT Gateway
- Internet Gateway
- Key-Pair (SSH Key)
- SSH Security Group (needed to ssh into the demo instance)
- AWS EC2 Instance


There will be two workflows for use in Github Actions. One is automated by a push to the main branch and the other is a manual workflow activated once we are done with the demo:

## Prerequisites: 

- Preferred Linux OS (I am using Manjaro for this demo)
- AWS Account & Command Line Interface w/ access key and secrets
	- You may use hard coded AWS credentials in your `.aws` directory or utilize `aws-vault` to store access keys and secrets securely. 
- Git & Github Account (needed to fork repo & add secrets for Github Actions)
- Basic knowledge of Docker and web appd
- Text Editor or IDE such as VSCode (needed to edit resources for Github Actions)
## Fork The repo & Set up Secrets: 

Log into your Github account and search for aws-terraform-practice. 

Click on fork: 

![[fork1.png]]
Set the name to whatever you'd like and click create fork.
![[fork.png]]

In the new forked repo, you can click Code and copy the URL to clone the repo to your Linux machine. 

![[code_url.png]]

Click on Settings > Security > Secrets > Actions

![[settings.png]]
Click New Repository Secret

![[secrets.png]]
Create the following Secrets : 

```
AWS_ACCESS_KEY_ID

AWS_PROFILE

AWS_REGION

AWS_SECRET_ACCESS_KEY

```

![[enter.png]]
Once they are all added, they should appear at the bottom of the page.

![[finish.png]]


In order to add the SSH Public key needed by the `ec2_instance.tf`  config file, we need to add a Secret called SSH_PUBLIC_KEY. 

Create an SSH Key on your local machine: 

```bash

ssh-keygen -t ed25519 -C "terraform demo" -f ~/.ssh/demo-key 

# lets output the public key for use in the SSH_PUBLIC_KEY Secret:

cat ~/.ssh/demo-key.pub

```

Create the SSH_PUBLIC_KEY Secret using the output of the `cat` command above. 

Navigate to the demo directory of the forked repo and open the the `ec2_instance.tf`  file to reference the location of the SSH Public key as it relates to the Github Runner (this will be in the current directory of the runner as shown in the deploy.yaml steps.)

```hcl
resource "aws_key_pair" "demo_ssh_key" {
  key_name   = var.key_name
  public_key = file("demo-key.pub")
}

```

Navigate to the github/workflows directory and open the deploy.yaml file: 

```yaml
      # On push to main, build or change infrastructure according to Terraform configuration files
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_REGION: ${{ secrets.AWS_REGION }}
        AWS_PROFILE: ${{ secrets.AWS_PROFILE }}
        SSH_PUBLIC_KEY: ${{ secrets.SSH_PUBLIC_KEY }}
      run: cd demo \ 
      cat $SSH_PUBLIC_KEY > demo-key.pub \
      terraform apply -var-file="../dev.auto.tfvars" -auto-approve
      
```