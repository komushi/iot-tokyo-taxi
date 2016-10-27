# 1. Local Deployment Spring Cloud Stream with CLI

## 1-1. Prerequisites

* RabbitMQ
* MongoDB with block-level polygon data of Tokyo - to convert coordinates to block names

### 1-1-1. Start RabbitMQ

```
rabbitmq-server
```

### 1-1-2. Start Geode

```
java -jar spring-boot-geode-server-0.0.1-SNAPSHOT.jar --properties.batchSize=5
```

## 1-1-3. Start MongoDB Server

[Polygon data in MongoDB](/solution/mongodb_data)

## 1-2. Start Spring Cloud Stream Apps
### 1-2-1. HTTP Source to retrieve taxi raw event data

```
java -jar http-source-rabbit-1.0.2.RELEASE.jar --spring.cloud.stream.bindings.output.destination=http_raw --server.port=9000
```

### 1-2-2. Geocoding Processor to handle the data from HTTP Source(2-2-1)

```
java -jar spring-cloud-stream-processor-geocoding-reverse-0.0.1-SNAPSHOT.jar --spring.cloud.stream.bindings.input.destination=http_raw --spring.cloud.stream.bindings.input.group=transform_geotuple --spring.cloud.stream.bindings.output.contentType=application/json --spring.cloud.stream.bindings.output.destination=transform_geotuple --properties.mongo.database=geojson --properties.mongo.collection=blocks --server.port=9100
```

### 1-2-3. GemFire Sink to send geocoded data(2-2-2) to Geode

```
java -jar gemfire-sink-rabbit-1.0.2.RELEASE.jar --gemfire.region.region-name=RegRaw --gemfire.json=true --gemfire.key-expression="payload.getField('uuid')" --spring.cloud.stream.bindings.input.destination=transform_geotuple --spring.cloud.stream.bindings.input.group=gemfire_rawjson --server.port=9200
```

### 1-2-4. GemFire Source to retrieve top route count data from Geode

```
java -jar gemfire-source-rabbit-1.0.2.RELEASE.jar --gemfire.cacheEventExpression=newValue --gemfire.region.region-name=RegRouteTopTen --spring.cloud.stream.bindings.output.destination=gemfire_topoute --server.port=9300
```

### 1-2-5. WebSocket Sink to retrieve top route count data(2-2-4) and send to Stomp

```
java -jar spring-cloud-stream-sink-stomp-0.0.1-SNAPSHOT.jar --logging.level.io.pivotal.spring.cloud.stream.sink=TRACE --spring.cloud.stream.bindings.input.destination=gemfire_topoute --spring.cloud.stream.bindings.input.group=1 --stomp.topic=toproute --stomp.withsockjs=true --server.port=9400
```

### 1-2-6. WebSocket Sink to retrieve geocoded data(2-2-3) and send to Stomp

```
java -jar spring-cloud-stream-sink-stomp-multi-topic-0.0.1-SNAPSHOT.jar --logging.level.io.pivotal.spring.cloud.stream.sink=INFO --spring.cloud.stream.bindings.input.destination=transform_geotuple --spring.cloud.stream.bindings.input.group=1 --stomp.withsockjs=true --stomp.topicPath=/dropoffDistrictCode --server.port=9210
```

### 1-2-7. GemFire Source to retrieve dropoff district count data from Geode

```
java -jar gemfire-source-rabbit-1.0.2.RELEASE.jar --gemfire.cacheEventExpression=newValue --gemfire.region.region-name=RegDropoffDistrictTop --spring.cloud.stream.bindings.output.destination=gemfire_topdropoff --server.port=9310
```

### 1-2-8. WebSocket Sink to send dropoff district count data(2-2-7) to Stomp

```
java -jar spring-cloud-stream-sink-stomp-0.0.1-SNAPSHOT.jar --logging.level.io.pivotal.spring.cloud.stream.sink=TRACE --spring.cloud.stream.bindings.input.destination=gemfire_topdropoff --spring.cloud.stream.bindings.input.group=1 --stomp.topic=topdropoff --stomp.withsockjs=true --server.port=9410
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

