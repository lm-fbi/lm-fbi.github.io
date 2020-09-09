---
layout: default
title: Amundsen
parent: Deployments
nav_order: 2
---
# Lyft's Amundsen on AWS

We are hosting [Lyft's Amundsen](https://www.amundsen.io/amundsen/) on an AWS EC2 instance with a simple docker stack.

## Set-Up of the EC2 Instance

```shell
# bash
aws ec2 run-instances \
    --image-id ami-linux2 \
    --count 1 \
    --instance-type m5.large \
    --key-name "Richard Work" \
    --security-group-ids sg-903004f8 \
    --subnet-id subnet-6e7f829e
```

## Set-Up of the Docker Stack

Every step should be done with su permissions. Install and set-up docker, git and tmux:

```shell
sudo su
yum update -y
yum install -y docker git tmux
service docker start
sudo usermod -aG docker ec2-user
```

Set-up Docker Compose:

```shell
curl -L "https://github.com/docker/compose/releases/download/1.26.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
sysctl -w vm.max_map_count=262144
cd /
git clone --recursive https://github.com/amundsen-io/amundsen.git
cd amundsen/
# docker-compose -f docker-amundsen.yml up
```

`sysctl -w vm.max_map_count=262144` is used to avoid the common docker compose error:  
`es_amundsen | [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]`
