# 1. Docker Deployment with Spring Cloud Stream

## 1-1. Prerequisites

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
docker run -p 40404:40404 komushi/spring-boot-geode-server:1.0.0.APACHE-GEODE-INCUBATING-M2 --properties.useJmx=false
```

## 1-1-3. Start MongoDB Server

[Polygon data in MongoDB](/solution/mongodb_data)

## 1-2. Start Spring Cloud Stream Apps
### 1-2-1. HTTP Source to retrieve taxi raw event data

```
docker run -p 9000:8080 springcloudstream/http-source-rabbit:1.0.2.RELEASE --spring.cloud.stream.bindings.output.destination=http_raw --spring.rabbitmq.host=192.168.0.73 --spring.rabbitmq.port=5672 --spring.rabbitmq.username=test --spring.rabbitmq.password=test
```

### 1-2-2. Geocoding Processor to handle the data from HTTP Source(1-2-1)

```
docker run komushi/spring-cloud-stream-processor-geocoding-reverse --properties.mongo.hostName=192.168.11.104 --properties.mongo.port=27017 --properties.mongo.database=geojson --properties.mongo.collection=blocks --properties.mongo.user=root --properties.mongo.password=keepitsimple --spring.cloud.stream.bindings.input.destination=http_raw --spring.cloud.stream.bindings.input.group=transform_geotuple --spring.cloud.stream.bindings.output.contentType=application/json --spring.cloud.stream.bindings.output.destination=transform_geotuple --spring.rabbitmq.host=192.168.11.104 --spring.rabbitmq.port=5672 --spring.rabbitmq.username=test --spring.rabbitmq.password=test
```

### 1-2-3. GemFire Sink to send geocoded data(1-2-2) to Geode

```
docker run springcloudstream/gemfire-sink-rabbit:1.0.2.RELEASE --gemfire.pool.connect-type=server --gemfire.pool.host-addresses=192.168.11.104:40404 --gemfire.region.region-name=RegRaw --gemfire.json=true --gemfire.key-expression="payload.getField('uuid')" --spring.cloud.stream.bindings.input.destination=transform_geotuple --spring.cloud.stream.bindings.input.group=gemfire_rawjson --spring.rabbitmq.host=192.168.11.104 --spring.rabbitmq.port=5672 --spring.rabbitmq.username=test --spring.rabbitmq.password=test
```

### 1-2-4. GemFire Source to retrieve top route count data from Geode

```
docker run springcloudstream/gemfire-source-rabbit:1.0.2.RELEASE --gemfire.pool.connect-type=server --gemfire.pool.host-addresses=192.168.11.104:40404 --gemfire.cacheEventExpression=newValue --gemfire.region.region-name=RegRouteTopTen --spring.cloud.stream.bindings.output.destination=gemfire_topoute --spring.rabbitmq.host=192.168.11.104 --spring.rabbitmq.port=5672 --spring.rabbitmq.username=test --spring.rabbitmq.password=test
```

### 1-2-5. WebSocket Sink to retrieve top route count data(1-2-4) and send to Stomp

```
docker run -p 9400:8080 komushi/spring-cloud-stream-sink-stomp --stomp.topic=toproute --stomp.withsockjs=true --spring.cloud.stream.bindings.input.destination=gemfire_topoute --spring.cloud.stream.bindings.input.group=1 --spring.rabbitmq.host=192.168.11.104 --spring.rabbitmq.port=5672 --spring.rabbitmq.username=test --spring.rabbitmq.password=test
```

### 1-2-6. WebSocket Sink to retrieve geocoded data(1-2-3) and send to Stomp

```
docker run -p 9210:8080 komushi/spring-cloud-stream-sink-stomp-multi-topic --stomp.withsockjs=true --stomp.topic-path=/dropoffDistrictCode --spring.cloud.stream.bindings.input.destination=transform_geotuple --spring.cloud.stream.bindings.input.group=1 --spring.rabbitmq.host=192.168.11.104 --spring.rabbitmq.port=5672 --spring.rabbitmq.username=test --spring.rabbitmq.password=test
```

### 1-2-7. GemFire Source to retrieve dropoff district count data from Geode

```
docker run springcloudstream/gemfire-source-rabbit:1.0.2.RELEASE --gemfire.pool.connect-type=server --gemfire.pool.host-addresses=192.168.11.104:40404 --gemfire.cacheEventExpression=newValue --gemfire.region.region-name=RegDropoffDistrictTop --spring.cloud.stream.bindings.output.destination=gemfire_topdropoff --spring.rabbitmq.host=192.168.11.104 --spring.rabbitmq.port=5672 --spring.rabbitmq.username=test --spring.rabbitmq.password=test
```

### 1-2-8. WebSocket Sink to send dropoff district count data(1-2-7) to Stomp

```
docker run -p 9410:8080 komushi/spring-cloud-stream-sink-stomp --logging.level.io.pivotal.spring.cloud.stream.sink=TRACE --stomp.topic=topdropoff --stomp.withsockjs=true --spring.cloud.stream.bindings.input.destination=gemfire_topdropoff --spring.cloud.stream.bindings.input.group=1 --spring.rabbitmq.host=192.168.11.104 --spring.rabbitmq.port=5672 --spring.rabbitmq.username=test --spring.rabbitmq.password=test
```

# 2. Visualization

## 2-1. Start GUI

```
cd ng-topten-taxi-zones
live-server
```

## 2-2. Start browser to monitor real-time events

東京タクシー降車イベント
http://localhost:8080/#/tokyo23TaxiEvents

![Districts](https://rawgit.com/komushi/iot-tokyo-taxi/master/image/districts.png)

## 2-3. Start browser to monitor real-time aggregated events

東京トップ１０タクシー・ルート
http://localhost:8080/#/tokyo23TopRoutes

![Top Routes](https://rawgit.com/komushi/iot-tokyo-taxi/master/image/top_routes.png)

# 3. Sending data

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

