---
layout: post
published: true
title: Podman Kafka Connect custom image build and run container
date: 2020-03-26 01:28
category: kafka
author: Swapan Chakrabarty
tags: [kafka, docker, kafka-connect, containers, podman]
summary: kafka in containers with Podman instead of Docker
---

## one command to create a Kafka connect image with mysql jdbc driver

```bash
mkdir ~/kafka-connect ;\
cd kafka-connect ;\
wget -q "http://search.maven.org/remotecontent?filepath=mysql/mysql-connector-java/8.0.18/mysql-connector-java-8.0.18.jar" -O mysql-connector-java-8.0.18.jar ;\
cat >> Dockerfile << EOF
FROM confluentinc/cp-kafka-connect
ARG JDBC_JAR_FILE=mysql-connector-java-8.0.18.jar
ARG CON_PATH=/usr/share/java/kafka
COPY ${JDBC_JAR_FILE} ${CON_PATH}/${JDBC_JAR_FILE}
EOF
podman build -t swapanc/kafka-connect-custom:latest . ;\
podman image ls
```

## run custom kafka connect container image

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
  swapanc/kafka-connect-custom:
  ```
