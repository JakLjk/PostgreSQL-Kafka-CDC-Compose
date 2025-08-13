# PostgreSQL-Kafka-CDC-Compose
[![Docker](https://img.shields.io/badge/docker-ready-blue)](#)
[![PostgreSQL](https://img.shields.io/badge/debezium%2Fpostgres-17-blue)](#)
[![Zookeeper](https://img.shields.io/badge/confluentinc%2Fcp--zookeeper-5.5.3-blue)](#)
[![Kafka](https://img.shields.io/badge/confluentinc%2Fcp--enterprise--kafka-5.5.3-blue)](#)
[![Debezium Connect](https://img.shields.io/badge/debezium%2Fconnect-1.4-blue)](#)
[![Schema Registry](https://img.shields.io/badge/confluentinc%2Fcp--schema--registry-5.5.3-blue)](#)


## General Information
Purpose of this stack is to create PostgreSQL database that will be able to stream changes in real time.

This can be utilised in spark, by defining readStream that will read changes that are happening in the PostgreSQL database

## Technologies Used
### 1. PostgreSQL (Debezium-enabled)
- Image: debezium/postgres:17
- Purpose: Acts as the source database that user can utilise to do anything that standard PostgreSQL DB can handle
- How it works: 
    - Database uses logical replication and Write-Ahead-Log (WAL) to expose changes that happenned in the database
    - Debezium reads changes from the WAL and emits them as Kafka messages
### 2. Zookeper
- Image: confluentinc/cp-zookeeper:5.5.3
- Purpose: Provides coordination services for the Kafka cluster
- How it works: 
    - Keeps track of Kafka broker metadata, topic configurations, and distributed locks
    - Required for older Kafka versions for leader election and configuration management

### 3. Kafka (message broker)
- Image: confluentinc/cp-enterprise-kafka:5.5.3
- Purpose: Acts as the central event streaming platform
- How it works:
    - Recieves CDC events from Debezium via Kafka Connect
    - Organizes data into topics (one topic per database table by default)
> `KAFKA_ADVERTISED_LISTENERS` must be updated to the current host ip to allow external clients to connect

### 4. Debezium (Kafka Connect)
- Image: debezium/connect:1.4
- Purpose: Runs Kafka Connect with Debezium's CDC connectors preinstalled
- How it works:
    - Kafka Connect is a framework for moving data in and out of Kafka
    - Debezium's PostgreSQL connector reads WAL logs and produces events to Kafka topics


## How to set up:

To initialise the stack go to the repository with .yaml file and use:
```bash
docker-compose up -d
```

In PostgreSQL's container bash you have to set REPLICA IDENTITY FULL on the table that you want to monitor:
```bash
ALTER TABLE <schema>.<table_name> REPLICA IDENTITY FULL
```

Register your conector in order to begin tracking changes in specific table 
Provided debezium.json can be used as template, and next sent to the debezium by curl:
```bash
curl -X POST http://<host_ip>:8083/connectors \
  -H "Content-Type: application/json" \
  --data @debezium.json
```

Use the command attached below to consume topic within terminal (to see in real time new records that appear in the queue). This command also can serve a purpose of creating topic in Kafka (It will create new topic if debezium did not produce any new rows yet and topic has not been initialised)

```bash
docker run --rm -it --network <docker_network_name> \
  confluentinc/cp-kafkacat kafkacat \
  -b kafka:9092 \
  -C \
  -t <postgres_container_name>.<schema>.<table_name> \
  -o beginning
```