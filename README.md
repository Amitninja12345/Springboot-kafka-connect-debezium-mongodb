# Kafka Connect: Debezium, MongoDB and Spring Boot Demo

Spring Boot application demonstrating using Kafka Connect for Change Data Capture (CDC) to publish outbound events to Kafka from MongoDB.

 Using Change Data Capture (CDC) with Debezium to stream events from the NoSQL MongoDB database to the Kafka messaging broker.

Demo steps breakdown:

Build Spring Boot application with Java 17:
```
mvn clean install
```

Start Docker containers:
```
docker-compose up -d
```

Initiate the replica set on the MongoDB docker container:
```
docker exec -it mongodb mongosh --eval "rs.initiate({_id:'docker-rs', members: [{_id:0, host: 'mongodb'}]})"
```
The status of the replica set can be viewed with:
```
docker exec -it mongodb mongosh --eval "rs.status()"
```
This should show the single replica set, marked as the `PRIMARY` member.

Check status of Kafka Connect:
```
curl localhost:8083
```

List registered connectors (empty array initially):
```
curl localhost:8083/connectors
```

Register connector:
```
curl -i -X POST localhost:8083/connectors -H "Content-Type: application/json" -d @./connector/debezium-mongodb-source-connector.json
```
The response should show the connector was successfully registered:
```
HTTP/1.1 201 Created
```

List registered connectors:
```
curl localhost:8083/connectors
["debezium-mongodb-source-connector"]
```

Start Spring Boot application:
```
java -jar target/kafka-connect-debezium-mongodb-1.0.0.jar
```

Start `kafka-console-consumer` on the Kafka docker container to listen for the CDC event:
```
docker exec -ti kafka kafka-console-consumer --topic mongodb.demo.items --bootstrap-server kafka:29092
```

In a terminal window use curl to submit a POST REST request to the application to create an item:
```
curl -i -X POST localhost:9001/v1/items -H "Content-Type: application/json" -d '{"name": "test-item"}'
```

A response should be returned with the 201 CREATED status code and the new item id in the Location header:
```
HTTP/1.1 201 
Location: 653d06f08faa89580090466e
```

The Spring Boot application should log the successful item persistence:
```
Item created with id: 653d06f08faa89580090466e
```

View the CDC event consumed by the `kafka-console-consumer` from Kafka:
```
{"schema":{"type":"struct","fields": [...] "after":"{\"_id\": \"654cecdc4356b26c4bac68af\",\"name\": \"test-item\",\"_class\": \"demo.domain.Item\"}"
```

Get the item that has been created using curl:
```
curl -i -X GET localhost:9001/v1/items/653d06f08faa89580090466e
```

A response should be returned with the 200 SUCCESS status code and the item in the response body:
```
HTTP/1.1 200 
Content-Type: application/json

{"id":"653d06f08faa89580090466e","name":"test-item"}
```

In a terminal window use curl to submit a PUT REST request to the application to update the item:
```
curl -i -X PUT localhost:9001/v1/items/653d06f08faa89580090466e -H "Content-Type: application/json" -d '{"name": "test-item-update"}'
```

A response should be returned with the 204 NO CONTENT status code:
```
HTTP/1.1 204 
```

The Spring Boot application should log the successful update of the item:
```
Item updated with id: 653d06f08faa89580090466e - name: test-item-update
```

A second CDC event representing the update should be consumed by the `kafka-console-consumer`.

Delete the item using curl:
```
curl -i -X DELETE localhost:9001/v1/items/653d06f08faa89580090466e
```

The Spring Boot application should log the successful deletion of the item:
```
Deleted item with id: 653d06f08faa89580090466e
```

A third CDC event representing the delete should be consumed by the `kafka-console-consumer`

Delete registered connector:
```
curl -i -X DELETE localhost:8083/connectors/debezium-mongodb-source-connector
```

Stop containers:
```
docker-compose down
```


Manual clean up (if left containers up):
```
docker rm -f $(docker ps -aq)
```

