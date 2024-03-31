## Youtube tutorial

https://www.youtube.com/watch?v=brj9DhEOVd0

https://docs.confluent.io/3.3.0/installation/docker/docs/tutorials/connect-avro-jdbc.html

## How to start

### 1. Build and start the containers

```
docker-compose up --build
```


### 2. create topics

```
docker exec broker kafka-topics --create --topic quickstart-avro-offsets --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181 ^
	&& docker exec broker kafka-topics --create --topic quickstart-avro-config --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181 ^
	&& docker exec broker kafka-topics --create --topic quickstart-avro-status --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181
```

Expected output

```
Created topic quickstart-avro-offsets.
Created topic quickstart-avro-config.
Created topic quickstart-avro-status.
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
Expected output
```
[2023-02-03 04:55:13,758] INFO REST resources initialized; server is started and ready to handle requests (org.apache.kafka.connect.runtime.rest.RestServer)
```

### 5. Check list connectors in kafka-connect worker

```
curl -s -X GET http://localhost:8083/connector-plugins
```
Expected output
```json
[{"class":"io.confluent.connect.activemq.ActiveMQSourceConnector","type":"source","version":"5.2.2"},{"class":"io.confluent.connect.elasticsearch.ElasticsearchSinkConnector","type":"sink","version":"5.2.2"},{"class":"io.confluent.connect.gcs.GcsSinkConnector","type":"sink","version":"5.0.1"},{"class":"io.confluent.connect.hdfs.HdfsSinkConnector","type":"sink","version":"5.2.2"},{"class":"io.confluent.connect.hdfs.tools.SchemaSourceConnector","type":"source","version":"2.2.1-cp1"},{"class":"io.confluent.connect.ibm.mq.IbmMQSourceConnector","type":"source","version":"5.2.2"},{"class":"io.confluent.connect.jdbc.JdbcSinkConnector","type":"sink","version":"5.2.2"},{"class":"io.confluent.connect.jdbc.JdbcSourceConnector","type":"source","version":"5.2.2"},{"class":"io.confluent.connect.jms.JmsSourceConnector","type":"source","version":"5.2.2"},{"class":"io.confluent.connect.s3.S3SinkConnector","type":"sink","version":"5.2.2"},{"class":"io.confluent.connect.storage.tools.SchemaSourceConnector","type":"source","version":"2.2.1-cp1"},{"class":"io.debezium.connector.mysql.MySqlConnector","type":"source","version":"0.9.5.Final"},{"class":"org.apache.kafka.connect.file.FileStreamSinkConnector","type":"sink","version":"2.2.1-cp1"},{"class":"org.apache.kafka.connect.file.FileStreamSourceConnector","type":"source","version":"2.2.1-cp1"},{"class":"org.apache.kafka.connect.integration.MonitorableSinkConnector","type":"sink","version":"some great version"},{"class":"org.apache.kafka.connect.integration.MonitorableSourceConnector","type":"source","version":"an entirely different version"},{"class":"org.apache.kafka.connect.runtime.TestSinkConnector","type":"sink","version":"some great version"},{"class":"org.apache.kafka.connect.runtime.TestSourceConnector","type":"source","version":"an entirely different version"},{"class":"org.apache.kafka.connect.runtime.WorkerTest$WorkerTestConnector","type":"unknown","version":"1.0"},{"class":"org.apache.kafka.connect.runtime.rest.resources.ConnectorPluginsResourceTest$ConnectorPluginsResourceTestConnector","type":"unknown","version":"1.0"}]
```

### 6. Create connector JDBC source connector

6.1 Send "source.json" file to kafka

```
curl -d @"source.json" -H "Content-Type: application/json" -X POST http://localhost:8083/connectors
```

Expected output
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

### Note that the config above will be translated become this sql query below (The sql is just an explaination, no need to run it)

```sql
SELECT * 
FROM `connect_test`.`test` 
WHERE `connect_test`.`test`.`modified` > ? 
	AND `connect_test`.`test`.`modified` < ? 
ORDER BY `connect_test`.`test`.`modified` ASC
```

### 7. Create mysql table 'test'

7.1 Go inside quickstart-mysql container

mysql -uroot -p

enter password confluent

show databases;

use connect_test;

```
create table test (
      id int not null auto_increment,
      name varchar(100),
      email varchar(100),
      department varchar(100),
      modified timestamp not null default current_timestamp,
      primary key (id)
     );
```
	
Insert data into db

```sql
INSERT INTO test (name, email, department) VALUES ('sheldon', 'sheldon@bigbang.com', 'physicist');
```

### 8. Create and check if the connector JDBC source - topic has been created

```
docker exec broker kafka-topics --create --topic quickstart-jdbc-test --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181
```

```
docker exec broker kafka-topics --describe --zookeeper zookeeper:2181 | findstr quickstart-jdbc-test
```

Expected output
```
Topic:quickstart-jdbc-test	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: quickstart-jdbc-test	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
```

### 9. Check the connector JDBC source status

```
curl -s -X GET http://localhost:8083/connectors/quickstart-jdbc-source/status
```

Expected output
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

### 10. Create connector file sink using topic quickstart-jdbc-test
```
curl -d @"source-sink.json" -H "Content-Type: application/json" -X POST http://localhost:8083/connectors
```
Expected output
```json
	{
	"name":"quickstart-avro-file-sink",
	"config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSinkConnector",
		"tasks.max":"1",
		"topics":"quickstart-jdbc-test",
		"file":"/tmp/files/jdbc-output.txt",
		"name":"quickstart-avro-file-sink"
		},
	"tasks":[],
	"type":"sink"
	}
```

This will create `sink/files/jdbc-output.txt` file inside the kafka-connect folder, and the text inside the file is
```
Struct{id=1,name=sheldon,email=sheldon@bigbang.com,department=physicist,modified=Fri Feb 03 05:17:30 UTC 2023}
```


### 11. Check the connector file sink status

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

### 12. Testing update data in source DB then check the sink files and elasticsearch

While listen for changes on sink file, insert new record to table `test` 

```
INSERT INTO test (name, email, department) VALUES ('sheldon2', 'sheldon2@bigbang.com', 'engineer');
```

expected new line in file `sink/files/jdbc-output.txt`
```
Struct{id=2,name=sheldon2,email=sheldon2@bigbang.com,department=engineer,modified=Sun Jul 07 16:58:35 UTC 2019}
```

# Your turn
1. Source (mysql) --> Sink (mongodb)
2. Source (mysql1) --> Sink (mysql2)

### Mongodb
- https://www.w3schools.com/python/python_mongodb_getstarted.asp

### Mongodb connector
- https://www.mongodb.com/docs/kafka-connector/current/quick-start/

### Necessary command
- To delete existing connector
``` 
curl -X DELETE http://localhost:8083/connectors/your-connector-name
```

### Necessary commands for exercise mysql1 --> mysql2

Don't forget to change to yr IP address and don't use NIDA network

```
curl -d @"sink2.json" -H "Content-Type: application/json" -X POST http://localhost:8083/connectors

curl -X DELETE http://localhost:8083/connectors/quickstart-avro-mysql-sink

curl -s -X GET http://localhost:8083/connectors/quickstart-avro-mysql-sink/status
```

### Real-time dataset
- https://archive.ics.uci.edu/dataset/791/metropt+3+dataset
