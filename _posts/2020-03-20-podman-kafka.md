---
layout: post
published: true
title: Podman Kafka RHEL 8
date: 2020-03-20 03:28
category: containers
author: Swapan Chakrabarty
tags: [openshift,minishift,codeready containers,docker,podman]
summary: podman kafka instead of Docker
---

### clean-up containers

```bash
bash << EOF
podman stop $(podman ps -a -q)
podman rm $(podman ps -a -q)
podman rmi $(podman images -a -q)
podman volume prune -f
podman ps -a
podman images -a
podman volume ls
EOF
```
   
### create podman zookeeper
```bash
podman run -d \
    --net=host \
    --name=zookeeper \
    -e ZOOKEEPER_CLIENT_PORT=32181 \
    -e ZOOKEEPER_TICK_TIME=2000 \
    confluentinc/cp-zookeeper:5.3.1-1
```   
   
### create kafka broker 
```bash
podman run -d \
    --net=host \
    --name=kafka \
    -e KAFKA_ZOOKEEPER_CONNECT=localhost:32181 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:29092 \
    -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
    -p 29092:29092 \
    confluentinc/cp-kafka:5.3.1-1
```   
   
### create schema registry
```bash
podman run -d \
  --net=host \
  --name=schema-registry \
  -e SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=localhost:32181 \
  -e SCHEMA_REGISTRY_HOST_NAME=localhost \
  -e SCHEMA_REGISTRY_LISTENERS=http://localhost:8081 \
  -p 8081:8081 \
  confluentinc/cp-schema-registry:5.3.1-1
```   
   
### create topics
```bash
podman run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:5.3.1-1 \
  kafka-topics --create --topic quickstart-avro-offsets --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
```
```bash
podman run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:5.3.1-1 \
  kafka-topics --create --topic quickstart-avro-config --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
```   
```bash
podman run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:5.3.1-1 \
  kafka-topics --create --topic quickstart-avro-status --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
```   
   
### verify topic creation
```bash
podman run \
   --net=host \
   --rm \
   confluentinc/cp-kafka:5.3.1-1 \
   kafka-topics --describe --zookeeper localhost:32181
```   
   
### start mysql db container
```bash
podman run -d \
    --name quickstart-mysql \
    --net=host \
    -e MYSQL_USER=confluent \
    -e MYSQL_PASSWORD=confluent \
    -e MYSQL_DATABASE=connect_test \
    -p 3306:3306 \
    rhscl/mysql-57-rhel7
```   
   
### build a custom Kafka-connect Docker image   
- create a Kafka-connect folder to hold the dockerfile and mysql driver
```bash
# create a Kafka-connect folder to hold the dockerfile and mysql driver
mkdir kafka-connect
cd kafka-connect
#
# create a dockerfile
cat >> Dockerfile << EOF
FROM confluentinc/cp-kafka-connect
ARG JDBC_JAR_FILE=mysql-connector-java-8.0.18.jar
ARG CON_PATH=/usr/share/java/kafka
COPY ${JDBC_JAR_FILE} ${CON_PATH}/${JDBC_JAR_FILE}
EOF
#
# get the latest mysql jdbc driver
wget "http://search.maven.org/remotecontent?filepath=mysql/mysql-connector-java/8.0.19/mysql-connector-java-8.0.19.jar" -O mysql-connector-java-8.0.19.jar
```   
   
- Build the custom kafka-connect image
```bash
cd /root/kafka-connect  ;\
podman build -t swapanc/kafka-connect:latest . ;\
podman image ls
```   
   
- connect to the mysql container and configure database
```bash
podman exec -it quickstart-mysql bash
```   
```sql
mysql -u confluent -p confluent
CREATE DATABASE IF NOT EXISTS connect_test;
USE connect_test;
DROP TABLE IF EXISTS test;
CREATE TABLE IF NOT EXISTS test (
  id serial NOT NULL PRIMARY KEY,
  name varchar(100),
  email varchar(200),
  department varchar(200),
  modified timestamp default CURRENT_TIMESTAMP NOT NULL,
  INDEX `modified_index` (`modified`)
);
INSERT INTO test (name, email, department) VALUES ('alice', 'alice@abc.com', 'engineering');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
exit;
```   
   
- start custom kafka connect connector
```bash
podman run -d \
  --name=kafka-connect-avro \
  --net=host \
  -p 28083:28083 \
  -e CONNECT_BOOTSTRAP_SERVERS=localhost:29092 \
  -e CONNECT_REST_PORT=28083 \
  -e CONNECT_GROUP_ID="quickstart-avro" \
  -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-avro-config" \
  -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-avro-offsets" \
  -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-avro-status" \
  -e CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_KEY_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_VALUE_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL="http://localhost:8081" \
  -e CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL="http://localhost:8081" \
  -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_REST_ADVERTISED_HOST_NAME="localhost" \
  -e CONNECT_LOG4J_ROOT_LOGLEVEL=DEBUG \
  -e CONNECT_PLUGIN_PATH=/usr/share/java,/etc/kafka-connect/jars \
  -p 2803:2803 \
  swapanc/kafka-connect
```

## single command
```bash
podman stop $(podman ps -a -q) ;\
podman rm $(podman ps -a -q) ;\
podman rmi $(podman images -a -q) ;\
podman volume prune -f ;\
podman ps -a ;\
podman images -a ;\
podman volume ls ;\
podman run -d \
    --net=host \
    --name=zookeeper \
    -e ZOOKEEPER_CLIENT_PORT=32181 \
    -e ZOOKEEPER_TICK_TIME=2000 \
    confluentinc/cp-zookeeper:5.3.1-1 ;\
podman run -d \
    --net=host \
    --name=kafka \
    -e KAFKA_ZOOKEEPER_CONNECT=localhost:32181 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:29092 \
    -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
    -p 29092:29092 \
    confluentinc/cp-kafka:5.3.1-1 ;\
podman run -d \
  --net=host \
  --name=schema-registry \
  -e SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=localhost:32181 \
  -e SCHEMA_REGISTRY_HOST_NAME=localhost \
  -e SCHEMA_REGISTRY_LISTENERS=http://localhost:8081 \
  -p 8081:8081 \
  confluentinc/cp-schema-registry:5.3.1-1 ;\
podman run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:5.3.1-1 \
  kafka-topics --create --topic quickstart-avro-offsets --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181 ;\
podman run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:5.3.1-1 \
  kafka-topics --create --topic quickstart-avro-config --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181 ;\
podman run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:5.3.1-1 \
  kafka-topics --create --topic quickstart-avro-status --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181 ;\
podman run \
   --net=host \
   --rm \
   confluentinc/cp-kafka:5.3.1-1 \
   kafka-topics --describe --zookeeper localhost:32181 ;\
podman run -d \
    --name quickstart-mysql \
    --net=host \
    -e MYSQL_USER=confluent \
    -e MYSQL_PASSWORD=confluent \
    -e MYSQL_DATABASE=connect_test \
    -p 3306:3306 \
    rhscl/mysql-57-rhel7 ;\
mkdir ~/kafka-connect ;\
cd kafka-connect ;\
wget -q "http://search.maven.org/remotecontent?filepath=mysql/mysql-connector-java/8.0.18/mysql-connector-java-8.0.18.jar" -O mysql-connector-java-8.0.18.jar ;\
cat >> Dockerfile << EOF
FROM confluentinc/cp-kafka-connect
ARG JDBC_JAR_FILE=mysql-connector-java-8.0.18.jar
ARG CON_PATH=/usr/share/java/kafka
COPY ${JDBC_JAR_FILE} ${CON_PATH}/${JDBC_JAR_FILE}
EOF
podman build -t swapanc/kafka-connect:latest . ;\
podman image ls
```   
   
- connect to the mysql container and configure database
```bash
podman exec -it quickstart-mysql bash
```   
```sql
mysql -u confluent -p confluent
CREATE DATABASE IF NOT EXISTS connect_test;
USE connect_test;
DROP TABLE IF EXISTS test;
CREATE TABLE IF NOT EXISTS test (
  id serial NOT NULL PRIMARY KEY,
  name varchar(100),
  email varchar(200),
  department varchar(200),
  modified timestamp default CURRENT_TIMESTAMP NOT NULL,
  INDEX `modified_index` (`modified`)
);
INSERT INTO test (name, email, department) VALUES ('alice', 'alice@abc.com', 'engineering');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
exit;
```   
   
- start custom kafka connect connector
```bash
podman run -d \
  --name=kafka-connect-avro \
  --net=host \
  -p 28083:28083 \
  -e CONNECT_BOOTSTRAP_SERVERS=localhost:29092 \
  -e CONNECT_REST_PORT=28083 \
  -e CONNECT_GROUP_ID="quickstart-avro" \
  -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-avro-config" \
  -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-avro-offsets" \
  -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-avro-status" \
  -e CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_KEY_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_VALUE_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL="http://localhost:8081" \
  -e CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL="http://localhost:8081" \
  -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_REST_ADVERTISED_HOST_NAME="localhost" \
  -e CONNECT_LOG4J_ROOT_LOGLEVEL=DEBUG \
  -e CONNECT_PLUGIN_PATH=/usr/share/java,/etc/kafka-connect/jars \
  -p 2803:2803 \
  swapanc/kafka-connect
```   
