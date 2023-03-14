---
# User change
title: "Install Redis on a Docker container"

weight: 6 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Install Redis on a single AWS Arm based instance 

You can deploy Redis on AWS Graviton processors using Terraform and Ansible. 

In this topic, you will deploy Redis on a Docker container, and in the next topic you will deploy Redis in a multi-node configuration. 

## Before you begin

You should have the prerequisite tools installed from the topic, [Install Redis on a single AWS Arm based instance](/learning-paths/server-and-cloud/redis/aws_deployment)

Use the same AWS access key ID and secret access key and the same SSH key pair.

## Create an AWS EC2 instance using Terraform

Using a text editor, save the code below to in a file called `main.tf`

Scroll down to see the information you need to change in `main.tf`

```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AAXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "redis-deployment" {
  ami = "ami-0ca2eafa23bc3dd01"
  instance_type = "t4g.small"
  key_name= "aws_key"
  vpc_security_group_ids = [aws_security_group.main.id]
}

resource "aws_security_group" "main" {
  name        = "main"
  description = "Allow TLS inbound traffic"

  ingress {
    description      = "Open redis connection port"
    from_port        = 6379
    to_port          = 6379
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  ingress {
    description      = "Allow ssh to instance"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }
}

output "Master_public_IP" {
  value = [aws_instance.redis-deployment.public_ip]
}

resource "local_file" "inventory" {
    depends_on=[aws_instance.redis-deployment]
    filename = "(your_current_directory)/hosts"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${aws_instance.redis-deployment.public_dns} ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "aws_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCUZXm6T6JTQBuxw7aFaH6gmxDnjSOnHbrI59nf+YCHPqIHMlGaxWw0/xlaJiJynjOt67Zjeu1wNPifh2tzdN3UUD7eUFSGcLQaCFBDorDzfZpz4wLDguRuOngnXw+2Z3Iihy2rCH+5CIP2nCBZ+LuZuZ0oUd9rbGy6pb2gLmF89GYzs2RGG+bFaRR/3n3zR5ehgCYzJjFGzI8HrvyBlFFDgLqvI2KwcHwU2iHjjhAt54XzJ1oqevRGBiET/8RVsLNu+6UCHW6HE9r+T5yQZH50nYkSl/QKlxBj0tGHXAahhOBpk0ukwUlfbGcK6SVXmqtZaOuMNlNvssbocdg1KwOH ubuntu@ip-172-31-XXXX-XXXX"
}
```
Make the changes listed below in `main.tf` to match your account settings.

1. In the `provider` section, update all 3 values to use your preferred AWS region and your AWS access key ID and secret access key.

2. (optional) In the `aws_instance` section, change the ami value to your preferred Linux distribution. The AMI ID for Ubuntu 22.04 on Arm is `ami-0ca2eafa23bc3dd01`. No change is needed if you want to use Ubuntu AMI. 

{{% notice Note %}}
The instance type is t4g.small. This an an Arm-based instance and requires an Arm Linux distribution.
{{% /notice %}}

3. In the `aws_key_pair` section, change the `public_key` value to match your SSH key. Copy and paste the contents of your aws_key.pub file to the `public_key` string. Make sure the string is a single line in the text file.

4. in the `local_file` section, change the `filename` to be the path to your current directory.

The hosts file is automatically generated and does not need to be changed, change the path to the location of the hosts file.


## Terraform Commands

Use Terraform to deploy the `main.tf` file.

### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command downloads the dependencies required for AWS.

```console
terraform init
```
    
The output should be similar to:

```console
Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/local...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/local v2.4.0...
- Installed hashicorp/local v2.4.0 (signed by HashiCorp)
- Installing hashicorp/aws v4.58.0...
- Installed hashicorp/aws v4.58.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```

A long output of resources to be created will be printed. 

### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan and create all AWS resources: 

```console
terraform apply
```      

Answer `yes` to the prompt to confirm you want to create AWS resources. 

The public IP address will be different, but the output should be similar to:

```console
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

Master_public_IP = [
  "3.135.226.118",
]
```

## Install Redis on a Docker container through Ansible
Install the Redis and the required dependencies. 

Using a text editor, save the code below to in a file called `playbook.yaml`. This is the YAML file for the Ansible playbook. 
```console
---
- hosts: all
  become: true
  become_user: root
  remote_user: ubuntu

  tasks:
    - name: Update the Machine and install docker dependencies
      shell: |
        apt update
        apt install -y ca-certificates curl gnupg lsb-release
    - name: Create directory
      file:
        path: /etc/apt/keyrings
        state: directory
    - name: Download docker gpg key and install docker
      shell: |
        curl -fsSL 'https://download.docker.com/linux/ubuntu/gpg' | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
        apt update
        apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    - name: Start and enable docker service
      service:
        name: docker
        state: started
        enabled: yes
    - name: Start redis container
      shell: docker run --name redis-container -p 6000:6379 -d redis
    - name: Set Authentication password
      shell: docker exec -it redis-container redis-cli CONFIG SET requirepass "{password}"
```
Replace {password} with your value.

To access the Redis server running inside the Docker container on port **6379**, we need to expose it to any available port on the machine using **-p {port_no_of_machine}:6379** argument. We need to replace **{port_no_of_machine}** with its respective value. In our example, we have used port number **6000**.

### Ansible Commands

Substitute your private key name, and run the playbook using the  `ansible-playbook` command:

```console
ansible-playbook playbook.yaml -i hosts --key-file aws_key
```

Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to:

```console
PLAY [all] *****************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************
The authenticity of host 'ec2-3-135-226-118.us-east-2.compute.amazonaws.com (172.31.30.40)' can't be established.
ED25519 key fingerprint is SHA256:uWZgVeACoIxRDQ9TrqbpnjUz14x57jTca6iASH3gU7M.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
ok: [ansible-target1]

TASK [Update the Machine and install docker dependencies] *************************************************************************************************************
changed: [ansible-target1]

TASK [Create directory] ***********************************************************************************************************************************************
changed: [ansible-target1]

TASK [Download docker gpg key and install docker] *********************************************************************************************************************
changed: [ansible-target1]

TASK [Start and enable docker service] ********************************************************************************************************************************
changed: [ansible-target1]

TASK [Start redis container] ******************************************************************************************************************************************
changed: [ansible-target1]

TASK [Set Authentication password] ************************************************************************************************************************************
changed: [ansible-target1]

PLAY RECAP ************************************************************************************************************************************************************
ansible-target1            : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Connecting to the Redis server from local machine

Execute the steps below connect to remote Redis server from local machine.
1. We need to install redis-tools to interact with redis-server.
```console
apt install redis-tools
```
2. Connect to redis-server through redis-cli.
```console
redis-cli -h <public-IP-address> -p 6379
```
The output will be:
```console
ubuntu@ip-172-31-38-39:~$ redis-cli -h 172.31.30.40 -p 6379
172.31.30.40:6379> 
```
3. Authorize Redis with the password set by us in playbook.yaml file
```console
172.31.30.40:6379> ping
(error) NOAUTH Authentication required.
172.31.30.40:6379> AUTH 123456789
OK
172.31.30.40:6379> ping
PONG
```
4. Try out commands in the redis-cli
```console
172.31.30.40:6379> set name test
OK
172.31.30.40:6379> get name
"test"
172.31.30.40:6379>
```
You have successfully installed Redis on a Docker container.

### Clean up resources

Run `terraform destroy` to delete all resources created.

```console
terraform destroy
```

Continue the Learning Path to deploy Redis on a single Azure instance.
