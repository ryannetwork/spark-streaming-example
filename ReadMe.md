#### Project setup

`git clone spark-streaming-example`

`cd spark-streaming-example`

`docker-compose up -d`

`sbt assembly`

Create a topic for incoming events

`docker exec broker kafka-topics --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 3 --topic incoming-events`

Create Cassandra key space and table

docker exec -it cassandra /bin/bash
cqlsh
CREATE KEYSPACE standalone
  WITH REPLICATION = { 
   'class' : 'SimpleStrategy', 
   'replication_factor' : 1 
  };
CREATE TABLE standalone.blocked_ips (
   ip text PRIMARY KEY, 
   request_count int, 
   click_view_ratio float, 
   categories_count int);

Add Kafka file source connector for incoming events

`curl --header "Content-Type: application/json" --request POST --data '{"name":"FileStreamSourceConnector","config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector","tasks.max":"1","topic":"incoming-events","key.converter":"org.apache.kafka.connect.storage.StringConverter","value.converter":"org.apache.kafka.connect.storage.StringConverter","file":"/usr/share/incoming-events/events.json"}}' \
http://localhost:8083/connectors`

Start events generator

`python3 ./src/main/python/botgen.py --file ./events/events.json --duration 5`

Start spark application

`docker exec -it spark /bin/bash`

`mkdir /tmp/spark-events`

`/usr/share/spark/bin/spark-submit \
--class StreamingApp \
--master local[2] \
/usr/share/streaming_task/target/scala-2.11/streaming_task-assembly-0.1.0-SNAPSHOT.jar`

Check the result in Cassandra

`docker exec -it cassandra /bin/bash`

`cqlsh`

`SELECT * FROM standalone.blocked_ips LIMIT 10;`   


#### Utility commands

Connect to created container

`docker-compose exec broker /bin/bash`

Get list of topics

`docker-compose exec broker kafka-topics --list --zookeeper zookeeper:2181`

Get the first 10 messages from a topic

`docker exec broker kafka-console-consumer --bootstrap-server broker:9092 --topic incoming-events --from-beginning --max-messages 10`

Reset offset for spark consumer

`docker-compose exec broker kafka-consumer-groups --bootstrap-server broker:9092 --group spark_consumer --topic incoming-events --reset-offsets --to-earliest --execute`
