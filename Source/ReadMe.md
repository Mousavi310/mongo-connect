First download the mongo connector plugin and uncompress it:

[https://www.confluent.io/hub/mongodb/kafka-connect-mongodb] (https://www.confluent.io/hub/mongodb/kafka-connect-mongodb)

Create docker-compose file. Set the volume path for the plugin.

``` docker


```

Configure replica set:

``` bash

docker-compose exec mongo1 /usr/bin/mongo --eval '''if (rs.status()["ok"] == 0) {
    rsconf = {
      _id : "rs0",
      members: [
        { _id : 0, host : "mongo1:27017", priority: 1.0 },
        { _id : 1, host : "mongo2:27017", priority: 0.5 },
        { _id : 2, host : "mongo3:27017", priority: 0.5 }
      ]
    };
    rs.initiate(rsconf);
}
rs.conf();'''

```

Make sure that plugins is installed:

curl localhost:8083/connector-plugins | jq


``` json
curl localhost:8083/connector-plugins | jq

[
  {
    "class": "com.mongodb.kafka.connect.MongoSinkConnector",
    "type": "sink",
    "version": "0.2"
  },
  {
    "class": "com.mongodb.kafka.connect.MongoSourceConnector",
    "type": "source",
    "version": "0.2"
  },
  {
    "class": "io.confluent.connect.gcs.GcsSinkConnector",
    "type": "sink",
    "version": "5.0.1"
  },
  {
    "class": "io.confluent.connect.storage.tools.SchemaSourceConnector",
    "type": "source",
    "version": "2.1.1-cp1"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "2.1.1-cp1"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "2.1.1-cp1"
  }
]

```

As you see above the mongo connector pluggin is available for use.


Assume you have a database named `MyDb` and a collection named `Products` I create following source-connector.json file:


Use following body for sending data to connector:



``` json

    curl -X POST -H "Content-Type: application/json" -d @source-connector.json http://localhost:8083/connectors | jq
    {
      "name": "products-connector",
      "config": {
        "connector.class": "com.mongodb.kafka.connect.MongoSourceConnector",
        "tasks.max": "1",
        "connection.uri": "mongodb://mongo1:27017,mongo2:27017,mongo3:27017",
        "mode": "incrementing",
        "database": "MyDb",
        "collection": "Products",
        "topic.prefix": "Events",
        "name": "products-connector",
        "value.converter.schemas.enable": "false"
      },
      "tasks": [],
      "type": "source"
    }



```

Now you can view the status of connector:

    curl http://localhost:8083/connectors/products-connector/status | jq

``` json
{
  "name": "products-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "connect:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "connect:8083"
    }
  ],
  "type": "source"
}

```


I insert some data to the MyDb database and Products collection. I import following `CSV` data in my Products collection:


    Id,Name,Price
    1,Hat,10
    2,Shirt,20
    3,Bag,25


Now we can view kafka topics:
    docker-compose exec broker bash
    kafka-topics --zookeeper zookeeper:2181 --list

    Events.MyDb.Products
    __confluent.support.metrics
    __consumer_offsets
    _confluent-metrics
    docker-connect-configs
    docker-connect-offsets
    docker-connect-status

As you see above a topic named Events.MyDb.Products is created.

Now consume data:

    kafka-console-consumer --bootstrap-server broker:9092 --from-beginning --topic Events.MyDb.Products

``` json
{"schema":{"type":"string","optional":false},"payload":"{\"_id\": {\"_data\": \"825D6009BC000000012B022C0100296E5A1004FA0979C4602442A786D5B33C814C08F446645F696400645D6009BCFCE7D6006B20360A0004\"}, \"operationType\": \"insert\", \"clusterTime\": {\"$timestamp\": {\"t\": 1566575036, \"i\": 1}}, \"fullDocument\": {\"_id\": {\"$oid\": \"5d6009bcfce7d6006b20360a\"}, \"Id\": \"1\", \"Name\": \"Hat\", \"Price\": \"10\"}, \"ns\": {\"db\": \"MyDb\", \"coll\": \"Products\"}, \"documentKey\": {\"_id\": {\"$oid\": \"5d6009bcfce7d6006b20360a\"}}}"}
{"schema":{"type":"string","optional":false},"payload":"{\"_id\": {\"_data\": \"825D6009BC000000022B022C0100296E5A1004FA0979C4602442A786D5B33C814C08F446645F696400645D6009BCFCE7D6006B20360B0004\"}, \"operationType\": \"insert\", \"clusterTime\": {\"$timestamp\": {\"t\": 1566575036, \"i\": 2}}, \"fullDocument\": {\"_id\": {\"$oid\": \"5d6009bcfce7d6006b20360b\"}, \"Id\": \"2\", \"Name\": \"Shirt\", \"Price\": \"20\"}, \"ns\": {\"db\": \"MyDb\", \"coll\": \"Products\"}, \"documentKey\": {\"_id\": {\"$oid\": \"5d6009bcfce7d6006b20360b\"}}}"}
{"schema":{"type":"string","optional":false},"payload":"{\"_id\": {\"_data\": \"825D6009BC000000032B022C0100296E5A1004FA0979C4602442A786D5B33C814C08F446645F696400645D6009BCFCE7D6006B20360C0004\"}, \"operationType\": \"insert\", \"clusterTime\": {\"$timestamp\": {\"t\": 1566575036, \"i\": 3}}, \"fullDocument\": {\"_id\": {\"$oid\": \"5d6009bcfce7d6006b20360c\"}, \"Id\": \"3\", \"Name\": \"Bag\", \"Price\": \"25\"}, \"ns\": {\"db\": \"MyDb\", \"coll\": \"Products\"}, \"documentKey\": {\"_id\": {\"$oid\": \"5d6009bcfce7d6006b20360c\"}}}"}


```