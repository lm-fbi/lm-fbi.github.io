---
layout: default
title: Amundsen
parent: Deployments
nav_order: 2
---
# Lyft's Amundsen on AWS EC2 with Docker

We are hosting [Lyft's Amundsen](https://www.amundsen.io/amundsen/) on an AWS EC2 instance with a simple docker stack. This deployment is based on the installation guide [Bootstrap a default version of Amundsen using Docker](https://github.com/amundsen-io/amundsen/blob/master/docs/installation.md) from [amundsen.io](https://www.amundsen.io/amundsen/installation/).

## Set-Up of the EC2 Instance

1. Create a security role for the Amundsen instance:

    ```shell
    $ aws ec2 create-security-group \
    --group-name amundsen-security-group \
    --description "Amundsen security group" \
    --vpc-id vpc-XXXXXXXX

    {
        "GroupId": "sg-1234567890abcdef0"
    }
    ```

    Replace `vpn-XXXXXXXX` with your VPC ID.

1. Define environment variables

    1. Set security group environment variable with GroupID from last response:

        ```shell
        export AMUNDSEN_SG=sg-1234567890abcdef0
        ```

    1. Set environment variable for SSH key and

        ```shell
        export AMUNDSEN_AWS_SSH_KEY_NAME=SSH-Key-Name
        ```

        Of course, use a key name for a SSH key in AWS you have access to.

    1. Set environment variable for subnet with a suitable value from the table below

        ```shell
        export AMUNDSEN_SUBNET=subnet-XXXXXXXX
        ```

        | AZ            | Subnet          |
        | ------------- | --------------- |
        | eu-central-1a | subnet-3df2ee56 |
        | eu-central-1b | subnet-b7eca7ca |
        | eu-central-1c | subnet-d6d58b9b |

    Add inbound rules for SSH and Amundsen Web UI access:

    ```shell
    $ aws ec2 authorize-security-group-ingress \
    --group-id $AMUNDSEN_SG \
    --ip-permissions IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges='[{CidrIp=0.0.0.0/0,Description="Allow SSH access for all"}]',Ipv6Ranges='[{CidrIpv6=::/0,Description="Allow SSH access for all"}]'

    $ aws ec2 authorize-security-group-ingress \
    --group-id $AMUNDSEN_SG \
    --ip-permissions IpProtocol=tcp,FromPort=5000,ToPort=5000,IpRanges='[{CidrIp=0.0.0.0/0,Description="Allow Amundsen Web UI access"}]'
    ```

1. Start one `m5.large` instance with a 64-bit (x86) Amazon Linux 2 AMI (HVM):

    ```shell
    $ aws ec2 run-instances \
    --image-id ami-08c148bb835696b45 \
    --count 1 \
    --instance-type m5.large \
    --key-name $AMUNDSEN_AWS_SSH_KEY_NAME \
    --security-group-ids $AMUNDSEN_SG \
    --subnet-id $AMUNDSEN_SUBNET \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Amundsen},{Key=system,Value=amundsen}]'
    ```

1. After the instance is booted, log into the new instance. Use the just set tag `Name:Amundsen` to find the instance and to get its public IP address:

    ```shell
    $ aws ec2 describe-instances \
    --filters Name=tag:Name,Values=Amundsen Name=instance-state-name,Values=running \
    --query "Reservations[].Instances[].{StartTime:LaunchTime, IP:PublicIpAddress, Type:InstanceType, Key:KeyName}"

    [
        {
            "StartTime": "2020-09-09T14:44:33+00:00",
            "IP": "123.123.123.123",
            "Type": "m5.large",
            "Key": "My Key"
        },
        {
            "StartTime": "2020-09-08T11:19:37+00:00",
            "IP": "234.234.234.234",
            "Type": "m5.large",
            "Key": "My Key"
        }
    ]

    $ export AMUNDSEN_IP=123.123.123.123
    ```

    ```shell
    $ ssh ec2-user@$AMUNDSEN_IP

        __|  __|_  )
        _|  (     /   Amazon Linux 2 AMI
        ___|\___|___|

    https://aws.amazon.com/amazon-linux-2/

    [ec2-user@ip-123-123-123-123 ~]$
    ```

## Set-Up of the Docker Stack

Every step should be done with super user permissions. Install and set-up docker, git and tmux:

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

`sysctl -w vm.max_map_count=262144` is used to avoid the common [docker compose error](https://github.com/amundsen-io/amundsen/issues/77#issuecomment-497895499):  
`es_amundsen | [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]`

## Related Articles

- [AWS Docs: Creating, configuring, and deleting security groups for Amazon EC2](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-sg.html)
- [AWS CLI Command Reference: EC2](https://docs.aws.amazon.com/cli/latest/reference/ec2/index.html#cli-aws-ec2)
- [Docker and Docker-Compose Setup on AWS EC2 Instance](https://medium.com/@khandelwal12nidhi/docker-setup-on-aws-ec2-instance-c670ff3d5f1b)
