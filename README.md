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
CONTAINER ID   IMAGE                                COMMAND                 CREATED          STATUS          PORTS                                                                                  NAMES
11d3dc8ff9c6   apache/doris:2.0.0_alpha-be-x86_64   "bash entry_point.sh"   19 minutes ago   Up 19 minutes   0.0.0.0:8041->8040/tcp, :::8041->8040/tcp                                              doris-be-01
b442f3ab54e7   apache/doris:2.0.0_alpha-be-x86_64   "bash entry_point.sh"   19 minutes ago   Up 19 minutes   0.0.0.0:8043->8040/tcp, :::8043->8040/tcp                                              doris-be-03
72dcab94a6a3   apache/doris:2.0.0_alpha-be-x86_64   "bash entry_point.sh"   19 minutes ago   Up 19 minutes   0.0.0.0:8042->8040/tcp, :::8042->8040/tcp                                              doris-be-02
8417f1ba3b4d   apache/doris:2.0.0_alpha-fe-x86_64   "bash init_fe.sh"       19 minutes ago   Up 19 minutes   0.0.0.0:8031->8030/tcp, :::8031->8030/tcp, 0.0.0.0:9031->9030/tcp, :::9031->9030/tcp   doris-fe-01
3c51890c2964   apache/doris:2.0.0_alpha-fe-x86_64   "bash init_fe.sh"       19 minutes ago   Up 19 minutes   0.0.0.0:8032->8030/tcp, :::8032->8030/tcp, 0.0.0.0:9032->9030/tcp, :::9032->9030/tcp   doris-fe-02
aa75071b05a7   apache/doris:2.0.0_alpha-fe-x86_64   "bash init_fe.sh"       19 minutes ago   Up 19 minutes   0.0.0.0:8033->8030/tcp, :::8033->8030/tcp, 0.0.0.0:9033->9030/tcp, :::9033->9030/tcp   doris-fe-03

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
    fruit varchar(50),
    price double
) ENGINE=OLAP
DISTRIBUTED BY HASH(fruit) BUCKETS 3;
```

Create a Routine Load Job

```
CREATE ROUTINE LOAD products.kafka_table ON kafka_table
COLUMNS TERMINATED BY ","
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
    "property.group.id" = "1",
    "property.client.id" = "2"
);
```

Query the topic

```
SELECT * FROM kafka_table;
```
