---
# User change
title: "Install Redis in a multi-node configuration"

weight: 7 # 1 is first, 2 is second, etc.

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

After generating the public and private keys, we will push our public key to the **authorized_keys** folder in **~/.ssh**. We will also create a security group that opens inbound port **22**(ssh). Also every Redis Cluster node requires two TCP connections open. The normal Redis TCP port used to serve clients, for example **6379**, plus the port obtained by adding 10000 to the data port, so **16379** in the example. Below is a Terraform file named **main.tf** which will do this for us. Here we are creating 6 instances.


```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "redis-deployment" {
  ami = "ami-0bc02c3c09aaee8ea"
  count = "6"
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
    description      = "Open port for Cluster bus"
    from_port        = 16379
    to_port          = 16379
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
[redis]

${aws_instance.redis-deployment[0].public_dns}
${aws_instance.redis-deployment[1].public_dns}
${aws_instance.redis-deployment[2].public_dns}
${aws_instance.redis-deployment[3].public_dns}
${aws_instance.redis-deployment[4].public_dns}
${aws_instance.redis-deployment[5].public_dns}

[all:vars]
host_key_checking=false
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "aws_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC/GFk2t5I2WOGWIP11kk9+sS2hwb+SuZV8b6KAi8IPR50pDjBXtBBt/8Apl+cyTmUjIlVxnyV6rS4sGVdKLC7SDNU8nl1SfDuh1HJRtlbMu8k+OmA3i9T/rihz2Qs9htkbSkdZ3bADCd5tcregPIht1bdQkjFK5zpbmiNHqIC1KJYIKfiwHMCLt+3ZQWr8iw1G19hHLbfpvDr0H/ewlrpMNG3StJSo6E2Jec6NZ09takFMl0a2r9Cej3bSQz5TuDnxWFDm1xk2svLojROnNeSH2sVx6UoPDpt05eniqgpYdMysYzxeOwS+qMHzR2IV2+0UoDFMxgcSgnhM36qlSk7H ubuntu@ip-172-XX-XX-XX"
}
```
**NOTE:-** Replace **public_key**, **access_key**, **secret_key**, and **key_name** with respective values.


### Terraform Commands

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/learning-paths/server-and-cloud/aws/terraform#terraform-commands) to deploy the **main.tf** file.

## Install Redis in a multi-node configuration using Ansible
To run Ansible, we have to create a **.yml** file, which is also known as **Ansible-Playbook**. The following playbook contains a collection of tasks that install Redis in a multi-node configuration (3 primary and 3 replica nodes). 

Here is the complete **deploy_redis.yml** file of Ansible-Playbook
```console
---
- name: Redis Cluster Install
  hosts: redis
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
         protected-mode no
         port 6379
         cluster-enabled yes
         cluster-config-file nodes.conf
         cluster-node-timeout 5000
         daemonize yes
         appendonly yes
      become_user: ubuntu
    - name: Stop redis-server
      shell: service redis-server stop
    - name: Start redis server with configuration files
      shell: redis-server redis.conf
      args:
        chdir: "/home/ubuntu/redis"
      become_user: ubuntu
```
**NOTE:-** Since the allocation of primary and replica nodes is random at the time of cluster creation, it is difficult to know which nodes are primary and which nodes are replica. Hence, for the multi-node configuration, we need to turn off **protected-mode**, which is enabled by default, so that we can connect to the primary and replica nodes. Also, the **bind address** is by default set to `127.0.0.1` due to which port 6379 becomes unavailable for binding with the public IP of the remote server. Thus, we set the bind configuration option to `0.0.0.0`.

To run a Playbook, we need to use the **ansible-playbook** command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace **{your_yml_file}**, **{your_inventory_file}** and **{path_to_private_key}** with your values.

![ansible-multi-final](https://user-images.githubusercontent.com/71631645/223932248-c8d38eec-0023-4c0f-9c94-f404e9588ba6.jpg)

Here is the output after the successful execution of the **ansible-playbook** command.

![ansible-multi-end](https://user-images.githubusercontent.com/71631645/223043499-2399df9b-7017-4f57-9735-e002cbab5f8f.jpg)

## Create a Redis cluster

After the Redis installation has been completed on all servers, lift the cluster up with the help of this command:

```console
redis-cli --cluster create {redis-deployment[0].public_ip}:6379 {redis-deployment[1].public_ip}:6379 {redis-deployment[2].public_ip}:6379 {redis-deployment[3].public_ip}:6379 {redis-deployment[4].public_ip}:6379 {redis-deployment[5].public_ip}:6379 --cluster-replicas 1
```
**NOTE:-** Replace **redis-deployment[n].public_ip** with their respective values.

Here is the output after the successful execution of the above command.

![cluster-start](https://user-images.githubusercontent.com/71631645/223050537-3a775d62-fefe-47e0-86c6-7cd0cb5ce542.jpg)
![cluster-end](https://user-images.githubusercontent.com/71631645/223050545-83928cf3-b9b4-4496-906e-8be4aac2e26c.jpg)

## Checking the status of the Redis Cluster

`cluster info` provides **info** style information about Redis Cluster vital parameters.
```console
redis-cli -c -h {redis-deployment[n].public_ip} -p 6379 cluter info
```
**Note:-** Replace **{redis-deployment[n].public_ip}** with the IP of any of the instances created.

![cluster-info](https://user-images.githubusercontent.com/71631645/224273631-0290c2bd-9348-4f51-ad5c-e1934e018e6c.jpg)

**cluster_state** is ok if the node is able to receive queries.


The `cluster nodes` command can be sent to any node in the cluster and provides the state of the cluster and the information for each node according to the local view the queried node has of the cluster.
```console
redis-cli -c -h {redis-deployment[n].public_ip} -p 6379 cluter nodes
```
![cluster-nodes-final](https://user-images.githubusercontent.com/71631645/224273685-2e196e03-8318-489c-b9d5-cea3810f5820.jpg)

## Connecting to Redis cluster from local machine

We can connect to remote Redis cluster from local machine using:

```console
redis-cli -c -h {redis-deployment[n].public_ip} -p 6379
```
The redis-cli will run in interactive mode. We can connect to any of the nodes, the command will get redirected to primary node.

![redis-cli](https://user-images.githubusercontent.com/71631645/224274757-1d705fc5-4f9f-41c4-a42c-2e0dfb3c5b98.jpg)
