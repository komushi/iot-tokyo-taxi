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
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_SKIP_SSL_VALIDATION true

cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_STREAM_MEMORY 512
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_STREAM_DISK 2048
cf set-env dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_STREAM_INSTANCES 1
```

```
cf start dataflow-server
```

## 1-2. Register Stream Apps

![Spring Cloud Stream Apps](https://rawgit.com/komushi/iot-tokyo-taxi/master/image/apps.png)

2 approaches to register stream apps:

### 1-2-1. Register with dashboard

| Name        | Type | URI |
| ------------- |---------------|---------------|
| http-source-rabbit | Source | http://192.168.22.1/http-source-rabbit-1.0.2.RELEASE.jar |
| gemfire-source-rabbit | Source | http://192.168.22.1/gemfire-source-rabbit-1.0.2.RELEASE.jar |
| gemfire-sink-rabbit | Sink | http://192.168.22.1/gemfire-sink-rabbit-1.0.2.RELEASE.jar |
| geocoding-processor-rabbit | Processor | http://192.168.22.1/spring-cloud-stream-processor-geocoding-reverse-0.2.0-SNAPSHOT.jar |
| stomp-sink-rabbit | Sink | http://192.168.22.1/spring-cloud-stream-sink-stomp-0.0.1-SNAPSHOT.jar |
| multi-topic-stomp-sink-rabbit | Sink | http://192.168.22.1/spring-cloud-stream-sink-stomp-multi-topic-0.0.1-SNAPSHOT.jar |

### 1-2-2. Optional: Register with Spring Cloud Dataflow Shell

```
app register --name http-source-rabbit --type source --uri 'http://192.168.22.1/http-source-rabbit-1.0.2.RELEASE.jar'

app register --name gemfire-sink-rabbit --type sink --uri 'http://192.168.22.1/gemfire-sink-rabbit-1.0.2.RELEASE.jar'

app register --name gemfire-source-rabbit --type source --uri 'http://192.168.22.1/gemfire-source-rabbit-1.0.2.RELEASE.jar'

app register --name geocoding-processor-rabbit --type processor --uri 'http://192.168.22.1/spring-cloud-stream-processor-geocoding-reverse-0.2.0-SNAPSHOT.jar'

app register --name stomp-sink-rabbit --type sink --uri 'http://192.168.22.1/spring-cloud-stream-sink-stomp-0.0.1-SNAPSHOT.jar'

app register --name multi-topic-stomp-sink-rabbit --type sink --uri 'http://192.168.22.1/spring-cloud-stream-sink-stomp-multi-topic-0.0.1-SNAPSHOT.jar'
```

### 1-3-1. Create on dashboard

```
http2multi-stomp=http-source-rabbit | geocoding-processor-rabbit --database=geojson --collection=blocks --host-name=192.168.12.22 --port=27017 | multi-topic-stomp-sink-rabbit --withsockjs=true --topic-path=/dropoffDistrictCode

http2gem=:http2multi-stomp.geocoding-processor-rabbit > gemfire-sink-rabbit --json=true --key-expression=payload.getField('uuid') --region-name=RegRaw

gem2stomp_top_district=gemfire-source-rabbit --cache-event-expression=newValue --region-name=RegDropoffDistrictTop | stomp-sink-rabbit --withsockjs=true --topic=topdropoff --port=9410

gem2stomp_top_route=gemfire-source-rabbit --cache-event-expression=newValue --region-name=RegRouteTopTen | stomp-sink-rabbit --withsockjs=true --topic=toproute --port=9400
```

### 1-3-2. Create on shell

```
stream create --name http2multi-stomp --definition "http-source-rabbit | geocoding-processor-rabbit --database=geojson --collection=blocks --host-name=192.168.12.22 --port=27017 | multi-topic-stomp-sink-rabbit --withsockjs=true --topic-path=/dropoffDistrictCode" --deploy

stream create --name http2gem --definition ":http2multi-stomp.geocoding-processor-rabbit > gemfire-sink-rabbit --json=true --key-expression=payload.getField('uuid') --region-name=RegRaw" --deploy

stream create --name gem2stomp_top_district --definition "gemfire-source-rabbit --cache-event-expression=newValue --region-name=RegDropoffDistrictTop | stomp-sink-rabbit --withsockjs=true --topic=topdropoff --port=9410" --deploy

stream create --name gem2stomp_top_route --definition "gemfire-source-rabbit --cache-event-expression=newValue --region-name=RegRouteTopTen | stomp-sink-rabbit --withsockjs=true --topic=toproute --port=9400" --deploy
```