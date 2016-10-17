# 1. Local Deployment with Spring Cloud Data Flow

## 1-1. Prerequisites

* Spring Cloud Dataflow Shell
* Spring Cloud Dataflow Server - Local
* RabbitMQ
* MongoDB with block-level polygon data of Tokyo - to convert coordinates to block names

### 1-1-1. Start RabbitMQ

```
rabbitmq-server
```

Or

```
brew services start rabbitmq
```

### 1-1-2. Start Geode

```
java -jar spring-boot-geode-server-0.0.1-SNAPSHOT.jar --properties.batchSize=5
```

### 1-1-3. Start Dataflow Server Local using RabbitMQ

```
java -jar spring-cloud-dataflow-server-local-1.0.0.RELEASE.jar --binder=rabbit
```

## 1-1-4. Start MongoDB Server

[Polygon data in MongoDB](/mongodb_data)

## 1-1-5. Start dashboard

```
http://localhost:9393/dashboard
```

## 1-2. Register Stream Apps

![Spring Cloud Stream Apps](https://rawgit.com/komushi/iot-tokyo-taxi/master/image/apps.png)

### 1-2-1. Register with dashboard

| Name        | Type | URI |
| ------------- |---------------|---------------|
| http-source-rabbit | Source | file:///temp/http-source-rabbit-1.0.2.RELEASE.jar |
| gemfire-source-rabbit | Source | file:///temp/gemfire-source-rabbit-1.0.2.RELEASE.jar |
| gemfire-sink-rabbit | Sink | file:///temp/gemfire-sink-rabbit-1.0.2.RELEASE.jar |
| geocoding-processor-rabbit | Processor | file:///temp/spring-cloud-stream-processor-geocoding-reverse-0.0.1-SNAPSHOT.jar |
| stomp-sink-rabbit | Sink | file:///temp/spring-cloud-stream-sink-stomp-0.0.1-SNAPSHOT.jar |
| multi-topic-stomp-sink-rabbit | Sink | file:///temp/spring-cloud-stream-sink-stomp-multi-topic-0.0.1-SNAPSHOT.jar |

### 1-2-2. Optional: Register with Spring Cloud Dataflow Shell

```
app register --name http-source-rabbit --type source --uri 'file:///temp/http-source-rabbit-1.0.2.RELEASE.jar'

app register --name gemfire-sink-rabbit --type sink --uri 'file:///temp/gemfire-sink-rabbit-1.0.2.RELEASE.jar'

app register --name gemfire-source-rabbit --type source --uri 'file:///temp/gemfire-source-rabbit-1.0.2.RELEASE.jar'

app register --name geocoding-processor-rabbit --type processor --uri 'file:///temp/spring-cloud-stream-processor-geocoding-reverse-0.0.1-SNAPSHOT.jar'

app register --name stomp-sink-rabbit --type sink --uri 'file:///temp/spring-cloud-stream-sink-stomp-0.0.1-SNAPSHOT.jar'

app register --name multi-topic-stomp-sink-rabbit --type sink --uri 'file:///temp/spring-cloud-stream-sink-stomp-multi-topic-0.0.1-SNAPSHOT.jar'
```

## 1-3. Create Stream

![Create Stream](https://rawgit.com/komushi/iot-tokyo-taxi/master/image/create_stream.png)

![Created Streams](https://rawgit.com/komushi/iot-tokyo-taxi/master/image/streams.png)

### 1-3-1. Create on dashboard

```
http2multi-stomp=http-source-rabbit --port=9000 | geocoding-processor-rabbit --database=geojson --collection=blocks | multi-topic-stomp-sink-rabbit --withsockjs=true --topic-path=/dropoffDistrictCode --port=9210

http2gem=:http2multi-stomp.geocoding-processor-rabbit > gemfire-sink-rabbit --json=true --key-expression=payload.getField('uuid') --region-name=RegRaw

gem2stomp_top_district=gemfire-source-rabbit --cache-event-expression=newValue --region-name=RegDropoffDistrictTop | stomp-sink-rabbit --withsockjs=true --topic=topdropoff --port=9410

gem2stomp_top_route=gemfire-source-rabbit --cache-event-expression=newValue --region-name=RegRouteTopTen | stomp-sink-rabbit --withsockjs=true --topic=toproute --port=9400
```

### 1-3-2. Create on shell

```
stream create --name http2multi-stomp --definition "http-source-rabbit --port=9000 | geocoding-processor-rabbit --database=geojson --collection=blocks | multi-topic-stomp-sink-rabbit --withsockjs=true --topic-path=/dropoffDistrictCode --port=9210" --deploy

stream create --name http2gem --definition ":http2multi-stomp.geocoding-processor-rabbit > gemfire-sink-rabbit --json=true --key-expression=payload.getField('uuid') --region-name=RegRaw" --deploy

stream create --name gem2stomp_top_district --definition "gemfire-source-rabbit --cache-event-expression=newValue --region-name=RegDropoffDistrictTop | stomp-sink-rabbit --withsockjs=true --topic=topdropoff --port=9410" --deploy

stream create --name gem2stomp_top_route --definition "gemfire-source-rabbit --cache-event-expression=newValue --region-name=RegRouteTopTen | stomp-sink-rabbit --withsockjs=true --topic=toproute --port=9400" --deploy
```

## 1-4. Sending data

```
java -jar FlatFileReader-0.1.0-jar-with-dependencies.jar --file=./data/sample_data_tokyo_23.csv --url http://localhost:9000 --mode=bulk --interval=100
```

| Fields        | Descriptions | Sample Data |
| ------------- |---------------|---------------|
|medallion | an md5sum of the identifier of the taxi - vehicle bound | 22D70BF00EEB0ADC83BA8177BB861991 |
|hack_license | an md5sum of the identifier for the taxi license| 3FF2709163DE7036FCAA4E5A3324E4BF |
|pickup_datetime | time when the passenger(s) were picked up| 2016-08-11 00:02:00 |
|dropoff_datetime | time when the passenger(s) were dropped off| 2016-08-11 00:05:00 |
|trip_time_in_secs | duration of the trip| 300 |
|trip_distance | trip distance in miles| 5400 |
|pickup_longitude | longitude coordinate of the pickup location| 139.706348 |
|pickup_latitude | latitude coordinate of the pickup location| 35.576938 |
|dropoff_longitude | longitude coordinate of the drop-off location| 139.76401 |
|dropoff_latitude | latitude coordinate of the drop-off location| 35.6651362 |
|payment_type | the payment method - credit card or cash| CSH |
|fare_amount | fare amount in currency| 2000 |
|surcharge | surcharge in currency| 0 |
|mta_tax | tax in currency| 160 |
|tip_amount | tip in currency| 0 |
|tolls_amount | bridge and tunnel tolls in currency| 0 |
|total_amount | total paid amount in currency| 2160 |

# 2. Local Deployment with CLI

## 2-1. Prerequisites

* RabbitMQ

### 2-1-1. Start RabbitMQ

```
rabbitmq-server
```

### 2-1-2. Start Geode

```
java -jar spring-boot-geode-server-0.0.1-SNAPSHOT.jar --properties.batchSize=5
```

## 2-2. Start Spring Cloud Stream Apps
### 2-2-1. HTTP Source to retrieve taxi raw event data

```
java -jar http-source-rabbit-1.0.2.RELEASE.jar --spring.cloud.stream.bindings.output.destination=http_raw --server.port=9000
```

### 2-2-2. Geocoding Processor to handle the data from HTTP Source(2-2-1)

```
java -jar spring-cloud-stream-processor-geocoding-reverse-0.0.1-SNAPSHOT.jar --spring.cloud.stream.bindings.input.destination=http_raw --spring.cloud.stream.bindings.input.group=transform_geotuple --spring.cloud.stream.bindings.output.contentType=application/json --spring.cloud.stream.bindings.output.destination=transform_geotuple --properties.mongo.database=geojson --properties.mongo.collection=blocks --server.port=9100
```

### 2-2-3. GemFire Sink to send geocoded data(2-2-2) to Geode

```
java -jar gemfire-sink-rabbit-1.0.2.RELEASE.jar --gemfire.region.region-name=RegRaw --gemfire.json=true --gemfire.key-expression="payload.getField('uuid')" --spring.cloud.stream.bindings.input.destination=transform_geotuple --spring.cloud.stream.bindings.input.group=gemfire_rawjson --server.port=9200
```

### 2-2-4. GemFire Source to retrieve top route count data from Geode

```
java -jar gemfire-source-rabbit-1.0.2.RELEASE.jar --gemfire.cacheEventExpression=newValue --gemfire.region.region-name=RegRouteTopTen --spring.cloud.stream.bindings.output.destination=gemfire_topoute --server.port=9300
```

### 2-2-5. WebSocket Sink to retrieve top route count data(2-2-4) and send to Stomp

```
java -jar spring-cloud-stream-sink-stomp-0.0.1-SNAPSHOT.jar --logging.level.io.pivotal.spring.cloud.stream.sink=TRACE --spring.cloud.stream.bindings.input.destination=gemfire_topoute --spring.cloud.stream.bindings.input.group=1 --stomp.topic=toproute --stomp.withsockjs=true --server.port=9400
```

## 2-2-6. WebSocket Sink to retrieve geocoded data(2-2-3) and send to Stomp

```
java -jar spring-cloud-stream-sink-stomp-multi-topic-0.0.1-SNAPSHOT.jar --logging.level.io.pivotal.spring.cloud.stream.sink=INFO --spring.cloud.stream.bindings.input.destination=transform_geotuple --spring.cloud.stream.bindings.input.group=1 --stomp.withsockjs=true --stomp.topicPath=/dropoffDistrictCode --server.port=9210
```

### 2-2-7. GemFire Source to retrieve dropoff district count data from Geode

```
java -jar gemfire-source-rabbit-1.0.2.RELEASE.jar --gemfire.cacheEventExpression=newValue --gemfire.region.region-name=RegDropoffDistrictTop --spring.cloud.stream.bindings.output.destination=gemfire_topdropoff --server.port=9310
```

### 2-2-8. WebSocket Sink to send dropoff district count data(2-2-7) to Stomp

```
java -jar spring-cloud-stream-sink-stomp-0.0.1-SNAPSHOT.jar --logging.level.io.pivotal.spring.cloud.stream.sink=TRACE --spring.cloud.stream.bindings.input.destination=gemfire_topdropoff --spring.cloud.stream.bindings.input.group=1 --stomp.topic=topdropoff --stomp.withsockjs=true --server.port=9410
```

## 2-3. Sending data

```
java -jar FlatFileReader-0.1.0-jar-with-dependencies.jar --file=./data/sample_data_tokyo_23.csv --url http://localhost:9000 --mode=bulk --interval=100
```

| Fields        | Descriptions | Sample Data |
| ------------- |---------------|---------------|
|medallion | an md5sum of the identifier of the taxi - vehicle bound | 22D70BF00EEB0ADC83BA8177BB861991 |
|hack_license | an md5sum of the identifier for the taxi license| 3FF2709163DE7036FCAA4E5A3324E4BF |
|pickup_datetime | time when the passenger(s) were picked up| 2016-08-11 00:02:00 |
|dropoff_datetime | time when the passenger(s) were dropped off| 2016-08-11 00:05:00 |
|trip_time_in_secs | duration of the trip| 300 |
|trip_distance | trip distance in miles| 5400 |
|pickup_longitude | longitude coordinate of the pickup location| 139.706348 |
|pickup_latitude | latitude coordinate of the pickup location| 35.576938 |
|dropoff_longitude | longitude coordinate of the drop-off location| 139.76401 |
|dropoff_latitude | latitude coordinate of the drop-off location| 35.6651362 |
|payment_type | the payment method - credit card or cash| CSH |
|fare_amount | fare amount in currency| 2000 |
|surcharge | surcharge in currency| 0 |
|mta_tax | tax in currency| 160 |
|tip_amount | tip in currency| 0 |
|tolls_amount | bridge and tunnel tolls in currency| 0 |
|total_amount | total paid amount in currency| 2160 |

# 3. Visualization

## 3-1. Start GUI

```
cd ng-topten-taxi-zones
live-server
```

## 3-2. Start GUI

東京タクシー降車イベント
http://localhost:8080/#/tokyo23TaxiEvents

![Districts](https://rawgit.com/komushi/iot-tokyo-taxi/master/image/districts.png)

## 3-3. Start GUI

東京トップ１０タクシー・ルート
http://localhost:8080/#/tokyo23TopRoutes

![Top Routes](https://rawgit.com/komushi/iot-tokyo-taxi/master/image/top_routes.png)