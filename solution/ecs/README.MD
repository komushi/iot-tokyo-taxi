## 1. ECS cluster creation preparation

### 1-1. Set the proper key and secret, use ap-northeast-1
```
$aws configure
```

### 1-2. Create a key-pair
```
$ aws ec2 create-key-pair --key-name dockerlab --query 'KeyMaterial' --output text > ./docker.pem
$ chmod 0400 ./docker.pem
```

### 1-3. Create a security group
```
$ aws ec2 create-security-group --group-name docker_SG --description "sg for docker cluster"
{
    "GroupId": "sg-9acfaefd"
}
```

Set up inbound rules of the security group
```
$ aws ec2 authorize-security-group-ingress --group-id sg-9acfaefd --protocol tcp --port 22 --cidr 0.0.0.0/0
$ aws ec2 authorize-security-group-ingress --group-id sg-9acfaefd --protocol tcp --port 27017 --cidr 0.0.0.0/0 --source-group sg-9acfaefd
$ aws ec2 authorize-security-group-ingress --group-id sg-9acfaefd --protocol tcp --port 5672 --cidr 0.0.0.0/0 --source-group sg-9acfaefd
$ aws ec2 authorize-security-group-ingress --group-id sg-9acfaefd --protocol tcp --port 15672 --cidr 0.0.0.0/0 --source-group sg-9acfaefd
```

### 1-4. create IAM roles for ECS with GUI
ecsInstanceRole with the following policies attached
* AmazonS3ReadOnlyAccess
* AmazonEC2ContainerServiceforEC2Role

ecsServiceRole with the following policies attached
* AmazonEC2ContainerServiceRole

## 2. Create a cluster
### 2-1. Create a cluster
```
$ aws ecs create-cluster --cluster-name iot-demo-cluster
```

## 2-2. Create a bucket to save the ECS config file for instances
```
$ aws s3api create-bucket --bucket docker-iot-demo-cluster
{
    "Location": "/docker-iot-demo-cluster"
}
```

### 2-3. Configure container agent for Amazon ECS-optimized AMI
```
$ aws s3 cp ./ecs.config s3://docker-iot-demo-cluster/ecs.config
$ aws s3 ls s3://docker-iot-demo-cluster
```

ecs.config
```
ECS_CLUSTER=iot-demo-cluster
```

Edit the 'copy-ecs-config-from-s3' file for container instances(Amazon ECS-optimized AMI)
```
#!/bin/bash

yum install -y aws-cli
aws s3 cp s3://docker-iot-demo-cluster/ecs.config /etc/ecs/ecs.config
```

### 2-4. Start 2 EC2 instances for container instances using the security group created in 1-3 and with the latest Amazon ECS-optimized AMI
```
$ aws ec2 run-instances --image-id ami-30bdce57 --count 2 --instance-type t2.medium --iam-instance-profile Name=ecsInstanceRole --key-name docker --security-group-ids sg-9acfaefd --user-data file://copy-ecs-config-from-s3
```

## 3. Start service by docker-compose with ecs-cli
```
$ ecs-cli configure --region ap-northeast-1 --access-key <access_key> --secret-key <secret> --cluster iot-demo-cluster
```

```
$ ecs-cli compose --file docker-compose-ecs.yml service up
```