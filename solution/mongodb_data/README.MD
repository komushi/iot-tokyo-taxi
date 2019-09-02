# 1. Import block-level polygon data of Tokyo to MongoDB

## 1-1. Start MongoDB Server
```
$ docker run -p 27017:27017 --name geojsonmongo mongo:3
```

## 1-2. Import data
```
$ docker cp tokyo_23_brief_mongo.json geojsonmongo:/tmp/tokyo_23_brief_mongo.json
$ docker exec geojsonmongo mongoimport -d geojson -c blocks --file /tmp/tokyo_23_brief_mongo.json
```

## 1-3. Add index and test
### 1-3-1. Login to the interactive mongo shell
```
$ docker exec -it geojsonmongo mongo
```

### 1-3-2. Add geometry index
```
use geojson

db.blocks.createIndex({ geometry: "2dsphere" })
```

### 1-3-3. If necessary create a user at geojson. Optional.
```
use geojson

db.createUser(
    {
      user: "root",
      pwd: "keepitsimple",
      roles: ["readWrite"]
    }
) 
```

## 1-3-4. Test geoIntersects query
```
db.blocks.find(
   {
     geometry: {
       $geoIntersects: {
          $geometry: {
             type: "Point" ,
             coordinates: [ 139.771699, 35.697283 ]
          }
       }
     }
   }
)
```