# 1. Prerequisites
* A working ECS Cluster or a Fargate Cluster(CloudFormation Template for Fargate Cluster is WIP).

## 1-1. ECS Cluster Wizard
* [Creating a ECS Cluster](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create_cluster.html)
* [ECSクラスターの作成](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/create_cluster.html)

## 1-2. Edit the EcsSecurityGroup's Inbound Rules

| Type            | Protocol | Port Range  | Source                      |
| --------------- | -------- | ----------- | --------------------------- |
| Custom TCP Rule | TCP      | 9000        | 0.0.0.0/0                   |
| Custom TCP Rule | TCP      | 9210        | 0.0.0.0/0                   |
| Custom TCP Rule | TCP      | 9400        | 0.0.0.0/0                   |
| Custom TCP Rule | TCP      | 9410        | 0.0.0.0/0                   |
| Custom TCP Rule | TCP      | 27017       | 0.0.0.0/0                   |
| Custom TCP Rule | TCP      | 40404       | 0.0.0.0/0                   |


# 2. Deployment
## 2-1. Deploy the Spring Cloud Stream system with CloudFormation Template

![Master Stack](https://cfn-template-group.s3.amazonaws.com/scs-iot/without_route53/master.yaml)

| Parameters      | Value                                           |
| --------------- | ----------------------------------------------- |
| Stack name      | The preferred name without spaces: scs or iot   |
| EcsClusterName  | The ECS Cluster Name created at #1              |
| VpcId           | The VPC created at #1                           |
| PublicSubnets   | The Public Subnets of the VPC created at #1     |
| SecurityGroup   | The ECS Security Group of the VPC created at #1 |


## 2-2. Deploy the Web Application to S3
Unzip ng-topten-taxi-zones.zip

### 2-2-1. Edit /config/config.json
Get the value of <LoadBalancerDns> from AlbStack's outpt
```
{
  "lbDns": "<LoadBalancerDns>"
}
```

### 2-2-2. Setting up a Static Website
* [Setting up a Static Website](https://docs.aws.amazon.com/AmazonS3/latest/dev/HostingWebsiteOnS3Setup.html)
* [静的ウェブサイトをセットアップする](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/dev/HostingWebsiteOnS3Setup.html)

```
$ aws s3 cp /aws_solution/ng-topten-taxi-zone/ s3://<your_bucket>/ --recursive
```

Open the static website after setup.

## 2-3. Start a new ECS task using flat-file-reader
Advanced Options - Container Overrides - Command Override
```
--url=http://<LoadBalancerDns>:9000
```
