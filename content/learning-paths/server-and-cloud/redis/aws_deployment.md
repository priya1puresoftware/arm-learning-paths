---
# User change
title: "Install Redis on a single AWS Arm based instance"

weight: 3 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need [an AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start). Create an account if needed.

Following tools are required on the computer you are using. Follow the links to install the required tools.
* [AWS CLI](/install-tools/aws-cli)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/install-tools/terraform)
* [Redis CLI](https://redis.io/docs/getting-started/installation/install-redis-on-linux/)

## Deploy AWS Arm based instance via Terraform

Before deploying AWS Arm based instance via Terraform, generate [Access keys](/learning-paths/server-and-cloud/aws/terraform#generate-access-keys-access-key-id-and-secret-access-key) and [key-pair using ssh keygen](/learning-paths/server-and-cloud/aws/terraform#generate-key-pairpublic-key-private-key-using-ssh-keygen).

After generating the public and private keys, we need to create an AWS Arm based instance. Then we will push our public key to the **authorized_keys** folder in **~/.ssh**. We will also create a security group that opens inbound ports **22**(ssh) and **6379**(Redis). Below is a Terraform file named **main.tf** which will do this for us.


```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AAXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "redis-deployment" {
  ami = "ami-0bc02c3c09aaee8ea"
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


resource "local_file" "inventory" {
    depends_on=[aws_instance.redis-deployment]
    filename = "inventory.txt"
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
**NOTE:-** Replace **public_key**, **access_key**, **secret_key**, and **key_name** with respective values.

Now, use the below Terraform commands to deploy the **main.tf** file.

### Terraform Commands

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/learning-paths/server-and-cloud/aws/terraform#terraform-commands) to deploy the main.tf file.

## Install Redis using Ansible
Ansible is a software tool that provides simple but powerful automation for cross-platform computer support.
Ansible allows you to configure not just one computer, but potentially a whole network of computers at once.
To run Ansible, we have to create a **.yml** file, which is also known as **Ansible-Playbook**. The following playbook contains a collection of tasks which install Redis on a single node.

Here is the complete **deploy_redis.yml** file of Ansible-Playbook
```console
---
- hosts: all
  become: true
  become_user: root
  remote_user: ubuntu

  tasks:
    - name: Update the Machine and install dependencies
      shell: |
             apt-get update -y
             curl -fsSL "https://packages.redis.io/gpg" | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
             echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" |  tee /etc/apt/sources.list.d/redis.list
             apt install -y redis-tools redis
    - name: Create directories
      file:
        path: "/home/ubuntu/redis"
        state: directory
      become_user: ubuntu
    - name: Create configuration files
      copy:
        dest: "/home/ubuntu/redis/redis.conf"
        content: |
          bind 0.0.0.0
          port 6379
          protected-mode yes
          cluster-enabled no
          daemonize yes
          appendonly no
      become_user: ubuntu
    - name: Stop redis-server
      shell: service redis-server stop
    - name: Start redis server with configuration files
      shell: redis-server redis.conf
      args:
        chdir: "/home/ubuntu/redis"
      become_user: ubuntu
    - name: Set Authentication password
      shell: redis-cli -p 6379 CONFIG SET requirepass "{password}"
      become_user: ubuntu
```
**NOTE:-** Replace **{password}** with respective value.

To run a Playbook, we need to use the **ansible-playbook** command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace **{your_yml_file}, {your_inventory_file}** and **{path_to_private_key}** with respective values.

Here is the output after the successful execution of the **ansible-playbook** command.

![ansible-aws-final-final](https://user-images.githubusercontent.com/71631645/223410377-af31650e-6c30-4f6b-968a-ac0a2e6b5db2.jpg)

## Connecting to Redis server from local machine

We can connect to remote Redis server from local machine. We need to use redis-tools to interact with redis-server.
```console
apt install redis-tools
```
```console
redis-cli -h {ansible_host} -p {port}
```
**NOTE:-** Get value of **{ansible_host}** from **inventory.txt** file and replace **{port}** with its respective value. 

The **redis-cli** will run in interactive mode. Before running any command, we need to authorize Redis with the **{password}** set by us in **deploy_redis.yml** file.
Otherwise, we will keep getting errors on running any command.

![redis-cli](https://user-images.githubusercontent.com/71631645/220876727-d04eab2c-3829-409a-86b2-5af91ee95661.jpg)
