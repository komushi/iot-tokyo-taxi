# 1. Import block-level polygon data of Tokyo to MongoDB

## 1-1. Create a cluster

```
$ aws ecs create-cluster --cluster-name iot-demo-cluster
```

## 1-2. Create a bucket to save the ECS config file for instances
```
$ aws s3api create-bucket --bucket docker-iot-demo-cluster
{
    "Location": "/docker-iot-demo-cluster"
}
```

```
$ aws s3 cp ./ecs.config s3://docker-iot-demo-cluster/ecs.config
$ aws s3 ls s3://docker-iot-demo-cluster
```

ecs.config
```
ECS_CLUSTER=iot-demo-cluster
```

## 1-3 edit the 'copy-ecs-config-from-s3' file for container instances
```
#!/bin/bash

yum install -y aws-cli
aws s3 cp s3://docker-iot-demo-cluster/ecs.config /etc/ecs/ecs.config
```



```
$ aws ec2 run-instances --image-id ami-30bdce57 --count 2 --instance-type t2.medium --iam-instance-profile Name=ecsInstanceRole --key-name docker --security-group-ids sg-713eb416 --user-data file://copy-ecs-config-from-s3
```

```
$ ecs-cli configure --region ap-northeast-1 --access-key <access_key> --secret-key <secret> --cluster iot-demo-cluster
```

```
$ ecs-cli compose --file docker-compose-ecs.yml service up
```