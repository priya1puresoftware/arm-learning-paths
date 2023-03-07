---
# User change
title: "Install Redis on a Docker container"

weight: 6 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need a cloud node or physical node with [Docker](/install-tools/docker/docker-engine) installed.

Following tools are required on the computer you are using. Follow the links to install the required tools.
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/install-tools/terraform)
* [Redis CLI](https://redis.io/docs/getting-started/installation/install-redis-on-linux/)

## Deploy AWS Arm based instance via Terraform

Before deploying AWS Arm based instance via Terraform, generate [Access keys](/learning-paths/server-and-cloud/aws/terraform#generate-access-keys-access-key-id-and-secret-access-key) and [key-pair using ssh keygen](/learning-paths/server-and-cloud/aws/terraform#generate-key-pairpublic-key-private-key-using-ssh-keygen).

Follow this [documentation](/learning-paths/server-and-cloud/redis/aws_deployment#deploy-aws-arm-based-instance-via-terraform) to deploy AWS Arm based instance via Terraform.


## Install Redis on a Docker container using Ansible
To run Ansible, we have to create a **.yml** file, which is also known as **Ansible-Playbook**. The following playbook contains a collection of tasks which install Redis on a Docker container.

Here is the complete **deploy_redis.yml** file of Ansible-Playbook
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
**NOTE:-** Replace **{password}** with respective value.

To access the Redis server running inside the Docker container on port **6379**, we need to expose it to any available port on the machine using **-p {port_no_of_machine}:6379** argument. We need to replace **{port_no_of_machine}** with its respective value. In our example, we have used port number **6000**.


To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace **{your_yml_file}** and **{path_to_private_key}** with respective values.

Here is the output after the successful execution of the **ansible-playbook** command.

![ansible-docker](https://user-images.githubusercontent.com/71631645/223415781-fe323bd0-a597-4b7f-87b4-8063cb02613e.jpg)

## Connecting to Redis server from local machine

Follow this [documentation](/learning-paths/server-and-cloud/redis/aws_deployment#connecting-to-redis-server-from-local-machine) to connect to the remote Redis server from our local machine.
