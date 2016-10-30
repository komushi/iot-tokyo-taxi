# 1. Cloud Foundry Deployment with Spring Cloud Data Flow

## 1-1. Prerequisites

* Spring Cloud Dataflow Server - Cloud Foundry (We use PCF Dev in this tutorial.)
* Redis, RabbitMQ - Included in PCF Dev
* CF CLI
* MongoDB with block-level polygon data of Tokyo - to convert coordinates to block names


```
cf login -a https://api.local2.pcfdev.io --skip-ssl-validation
```

```
cf create-service p-mysql 512mb mysql
```

```
cf create-service p-redis shared-vm redis
```

```
cf create-service p-rabbitmq standard rabbit
```

```
wget http://repo.spring.io/release/org/springframework/cloud/spring-cloud-dataflow-server-cloudfoundry/1.0.1.RELEASE/spring-cloud-dataflow-server-cloudfoundry-1.0.1.RELEASE.jar
wget http://repo.spring.io/release/org/springframework/cloud/spring-cloud-dataflow-shell/1.0.1.RELEASE/spring-cloud-dataflow-shell-1.0.1.RELEASE.jar
```

cf push dataflow-server -m 1G --no-start -p spring-cloud-dataflow-server-cloudfoundry-1.0.1.RELEASE.jar
cf bind-service dataflow-server redis
cf bind-service dataflow-server mysql


cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_URL https://api.local2.pcfdev.io
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_ORG pcfdev-org
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_SPACE pcfdev-space
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_DOMAIN local2.pcfdev.io
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_STREAM_SERVICES rabbit
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_USERNAME admin
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_PASSWORD admin
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_SKIP_SSL_VALIDATION false

cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_STREAM_MEMORY 512
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_STREAM_DISK 2048
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_STREAM_INSTANCES 1