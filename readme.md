
## How to start

### 1. Build and start the containers

```
docker-compose up --build
```


### 2. create topics

```
docker exec broker kafka-topics --create --topic quickstart-avro-offsets --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181 \
	&& docker exec broker kafka-topics --create --topic quickstart-avro-config --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181 \
	&& docker exec broker kafka-topics --create --topic quickstart-avro-status --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181
```

### 3. check the topics has been created

```
docker exec broker kafka-topics --describe --zookeeper zookeeper:2181
```

expected output

```
Topic:__confluent.support.metrics	PartitionCount:1	ReplicationFactor:1	Configs:retention.ms=31536000000
	Topic: __confluent.support.metrics	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
Topic:_schemas	PartitionCount:1	ReplicationFactor:1	Configs:cleanup.policy=compact
	Topic: _schemas	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
Topic:quickstart-avro-config	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: quickstart-avro-config	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
Topic:quickstart-avro-offsets	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: quickstart-avro-offsets	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
Topic:quickstart-avro-status	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: quickstart-avro-status	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
```

### 4. Check the kafka-connect worker with avro support is started

```
docker logs kafka-connect-avro | findstr started
```

### 5. Check list connectors in kafka-connect worker

```
curl -s -X GET http://localhost:8083/connectors-plugins
```

### 6. Create connector JDBC source connector

```
curl -X POST \
  http://localhost:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "quickstart-jdbc-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "tasks.max": 1,
    "connection.url": "jdbc:mysql://quickstart-mysql:3306/connect_test",
    "connection.user": "root",
    "connection.password": "confluent",
    "mode": "timestamp",
    "timestamp.column.name": "modified",
    "topic.prefix": "quickstart-jdbc-",
    "poll.interval.ms": 1000
  }
}'
```

output should be
```json
{
  "name": "quickstart-jdbc-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "tasks.max": "1",
    "connection.url": "jdbc:mysql://quickstart-mysql:3306/connect_test?user=root&password=confluent",
    "mode": "timestamp",
    "timestamp.column.name": "modified",
    "topic.prefix": "quickstart-jdbc-",
    "poll.interval.ms": "1000",
    "name": "quickstart-jdbc-source"
  },
  "tasks": [],
  "type": "source"
}
```

The config above will be translated become this sql query below (The sql is just an explaination, no need to run it)

```sql
SELECT * 
FROM `connect_test`.`test` 
WHERE `connect_test`.`test`.`modified` > ? 
	AND `connect_test`.`test`.`modified` < ? 
ORDER BY `connect_test`.`test`.`modified` ASC
```

![sourceJDBCMySQL-sinkFile](https://raw.githubusercontent.com/harryosmar/kafka-connect/master/doc/mysql-db.png)

### 7. Check if the connector JDBC source - topic has been created

```
docker exec broker kafka-topics --describe --zookeeper zookeeper:2181 | grep quickstart-jdbc-test
```

Output should be
```
Topic:quickstart-jdbc-test	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: quickstart-jdbc-test	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
```

### 8. Check the connector JDBC source status

```
curl -s -X GET http://localhost:8083/connectors/quickstart-jdbc-source/status
```

```json
{
  "name": "quickstart-jdbc-source",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect-avro:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "kafka-connect-avro:8083"
    }
  ],
  "type": "source"
}
```

### 9. Create connector file sink using topic quickstart-jdbc-test

```
curl -X POST \
  http://localhost:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "quickstart-avro-file-sink",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "tasks.max": "1",
    "topics": "quickstart-jdbc-test",
    "file": "/tmp/files/jdbc-output.txt"
  }
}'
```

```json
{
  "name": "quickstart-avro-file-sink",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "tasks.max": "1",
    "topics": "quickstart-jdbc-test",
    "file": "/tmp/files/jdbc-output.txt",
    "name": "quickstart-avro-file-sink"
  },
  "tasks": [],
  "type": "sink"
}
```

This will create `./sink/files/jdbc-output.txt`


### 10. Check the connector file sink status

```
curl -s -X GET http://localhost:8083/connectors/quickstart-avro-file-sink/status
```

```json
{
  "name": "quickstart-avro-file-sink",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect-avro:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "kafka-connect-avro:8083"
    }
  ],
  "type": "sink"
}
```

### 11. Testing update data in source DB then check the sink files and elasticsearch

While listen for changes on sink file, insert new record to table `test` 
```
INSERT INTO test (name, email, department) VALUES ('sheldon', 'sheldon@bigbang.com', 'physicist');
```

expected new line in file `./sink/files/jdbc-output.txt`
```
Struct{id=11,name=sheldon,email=sheldon@bigbang.com,department=physicist,modified=Sun Jul 07 16:58:35 UTC 2019}
```
