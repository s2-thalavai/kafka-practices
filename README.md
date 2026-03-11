# kafka-practices

## Kafka + Schema Registry + Kafka UI

```
          +-------------------+
          |    Kafka UI       |
          | (Web Dashboard)   |
          +---------+---------+
                    |
                    v
+-------------------------------------------+
|               Kafka Cluster               |
|   +--------+  +--------+  +--------+      |
|   |Kafka 1 |  |Kafka 2 |  |Kafka 3 |      |
|   +--------+  +--------+  +--------+      |
+-----------+---------+-----------+---------+
            |         |           |
            v         v           v
        +-----------------------------+
        |       Schema Registry       |
        +-----------------------------+
                    |
                    v
             Zookeeper Cluster
         +--------+--------+--------+
         | zoo1   | zoo2   | zoo3   |
         +--------+--------+--------+
```

--------------


# Start the Cluster

```bash
docker compose up -d
```

----------

# Kafka UI

Open browser:

```
http://localhost:8080
```

You can:

-   Create topics
    
-   Browse messages
    
-   View consumer groups
    
-   Produce messages
    
-   Monitor partitions
    

----------

# Schema Registry

Open:

```
http://localhost:8081
```

Example API:

```
GET /subjects
```

Example:

```
curl http://localhost:8081/subjects
```

----------

# Why Schema Registry is Important

The **Confluent Schema Registry** manages schemas for:

-   **Avro**
    
-   **JSON Schema**
    
-   **Protobuf**
    

Benefits:

| Feature                | Benefit                          |
| ---------------------- | -------------------------------- |
| Schema validation      | Prevents bad messages            |
| Compatibility checks   | Backward / forward compatibility |
| Central schema storage | All services share schema        |

----------


# Kafka UI Tool

The UI comes from **Kafka UI**.

Features:

-   Topic viewer
    
-   Message browser
    
-   Consumer lag monitoring
    
-   Schema registry integration
    
-   Produce messages from UI
    

----------

# Real Production Kafka Stack

Companies often run:

-   **Apache Kafka**
    
-   **Confluent Schema Registry**
    
-   **Prometheus**
    
-   **Grafana**
    
-   **Kubernetes**


----------


A **Kafka + Debezium + CDC pipeline** is a very common **real-time data architecture** used in microservices, data platforms, and event-driven systems.

It allows you to **capture database changes automatically and stream them to Kafka topics in real time**.


# Architecture Overview

![https://debezium.io/documentation/reference/stable/_images/debezium-architecture.png](https://debezium.io/documentation/reference/stable/_images/debezium-architecture.png)

![https://estuary.dev/static/222f991a2f7ace3cfc5a8c61ea62d503/d3153/02_Change_Data_Capture_Kafka_Change_Data_Capture_543562ecbc.png](https://estuary.dev/static/222f991a2f7ace3cfc5a8c61ea62d503/d3153/02_Change_Data_Capture_Kafka_Change_Data_Capture_543562ecbc.png)

![https://media.licdn.com/dms/image/v2/D4E12AQGtcGAZwPCLaw/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1713124638068?e=2147483647&t=S6Xbw1xwfszenkH564g1hFlqjet_2vrklYFblq1HP0I&v=beta](https://media.licdn.com/dms/image/v2/D4E12AQGtcGAZwPCLaw/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1713124638068?e=2147483647&t=S6Xbw1xwfszenkH564g1hFlqjet_2vrklYFblq1HP0I&v=beta)


Typical flow:

```
Database → Debezium Connector → Kafka → Consumers
```

Example:

```
MySQL → Debezium → Kafka Topic → Microservices / Data Lake / Analytics
```

----------

# Key Components

### 1. Database

Your source database.

Examples:

-   MySQL
    
-   PostgreSQL
    
-   MongoDB
    
-   SQL Server
    

Debezium reads **transaction logs** instead of polling.


## Example logs:

