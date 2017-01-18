# 1. Import block-level polygon data of Tokyo to MongoDB

## 1-1. Start MongoDB Server

```
mongod
```

Or

```
brew services start mongodb
```

Or

```
docker run -p 27017:27017 --name mongo mongo:3
```

## 1-2. Import data
```
mongoimport --db geojson --collection blocks --type json --file "tokyo_23_brief_mongo.json"
```

TODO: how to import data to mongo in docker

## 1-3. Add geometry index in mongo shell

```
use geojson

db.blocks.createIndex({ geometry: "2dsphere" })
```

### 1-3-1. If necessary create a user at geojson. Optional.
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

## 1-4. Test geoIntersects query in mongo shell

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