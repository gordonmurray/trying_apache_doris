# Trying out Apache Doris

![Static Badge](https://img.shields.io/badge/Just_testing-Not_production_ready-red)

Just trying out Apache Doris, after reading this post "[Empowering cyber security by enabling 7 times faster log analysis](https://doris.apache.org/blog/empowering-cyber-security-by-enabling-seven-times-faster-log-analysis)"

Before you start, run the following, it increases the maximum number of memory map areas a process can have, which is important for Apache Doris to handle large data sets and operations efficiently.

```
sudo sysctl -w vm.max_map_count=2000000
```

Then run the following to start a Doris front and back end:

```
docker compose up -d
```

You should see two containers running as a result, such as:

```
CONTAINER ID   IMAGE                                COMMAND                 CREATED       STATUS       PORTS     NAMES
c57efd4cdbf5   apache/doris:2.0.0_alpha-be-x86_64   "bash entry_point.sh"   1 hours ago   Up 1 hours             doris-be-1
bf82ce708150   apache/doris:2.0.0_alpha-fe-x86_64   "bash init_fe.sh"       1 hours ago   Up 1 hours             doris-fe-1
```

### Connect to the mySQL interface

```
mysql -h 127.0.0.1 -P 9030 -u root
```

### Doris Web UI

http://localhost:8030/home

Username: root and a blank password by default

### Create some Kafka content to consume

Create a topic

```
./bin/kafka-topics.sh --create --topic kafka_topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

Add some messages

```
echo "oranges,0.45" | ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic kafka_topic
```

Confirm there are some messages in the topic

```
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic kafka_topic --from-beginning
```

Create a table in Doris to start comsuming the kafka topic

```
CREATE TABLE kafka_table (
    fruit varchar,
    price double
) ENGINE=OLAP
DISTRIBUTED BY HASH(fruit) BUCKETS 10;
```

Create a Routine Load Job

```
CREATE ROUTINE LOAD db1.kafka_table ON kafka_table
COLUMNS TERMINATED BY ",",
PROPERTIES
(
    "desired_concurrent_number"="3",
    "max_batch_interval" = "20",
    "max_batch_rows" = "300000",
    "max_batch_size" = "209715200",
    "strict_mode" = "false",
    "format" = "csv"
)
FROM KAFKA
(
    "kafka_broker_list" = "localhost:9092",
    "kafka_topic" = "kafka_topic",
    "property.group.id" = "group_id",
    "property.client.id" = "client_id"
);
```

Query the topic

```
SELECT * FROM kafka_table;
```
