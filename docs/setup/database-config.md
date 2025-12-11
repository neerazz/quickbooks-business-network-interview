# Database Configuration Guide

## Overview

This guide covers the configuration of all databases used in the QuickBooks Business Network platform.

## PostgreSQL Configuration

### Connection Settings

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/quickbooks
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
```

### Database Schema

The following tables are created:

| Table | Purpose |
|-------|---------|
| businesses | Business entity storage |
| transactions | Transaction records |
| audit_logs | Audit trail for all operations |
| users | User accounts |
| entity_resolutions | Entity resolution results |

### Connection Pool (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 30000
      connection-timeout: 20000
      max-lifetime: 1800000
```

## Neo4j Configuration

### Connection Settings

```yaml
spring:
  neo4j:
    uri: bolt://localhost:7687
    authentication:
      username: neo4j
      password: password
```

### Graph Schema

```cypher
// Business nodes
CREATE CONSTRAINT business_id IF NOT EXISTS
FOR (b:Business) REQUIRE b.id IS UNIQUE;

// Relationship types
// - VENDOR_OF
// - CLIENT_OF
```

### Performance Tuning

```properties
# neo4j.conf
dbms.memory.heap.initial_size=512m
dbms.memory.heap.max_size=2g
dbms.memory.pagecache.size=512m
```

## Elasticsearch Configuration

### Connection Settings

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    connection-timeout: 5s
    socket-timeout: 30s
```

### Index Mappings

```json
{
  "mappings": {
    "properties": {
      "businessName": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "address": { "type": "text" },
      "category": { "type": "keyword" },
      "createdAt": { "type": "date" }
    }
  }
}
```

## Redis Configuration

### Connection Settings

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 1000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
```

### Cache Configuration

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes
      cache-null-values: false
```

## Kafka Configuration

### Broker Settings

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: quickbooks-consumer
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

### Topics

| Topic | Purpose |
|-------|---------|
| business.created | New business entity events |
| transaction.created | New transaction events |
| relationship.updated | Relationship change events |
| entity-resolution.pending | Pending resolutions |

## Milvus (Vector DB) Configuration

### Connection Settings

```yaml
milvus:
  host: localhost
  port: 19530
  collection: business_embeddings
  dimension: 768
```

## Docker Compose Setup

Start all databases:

```bash
docker-compose up -d postgres neo4j elasticsearch redis kafka zookeeper milvus
```

Verify services:

```bash
docker-compose ps
```

## Troubleshooting

### PostgreSQL Connection Issues

```bash
# Check if PostgreSQL is running
docker exec -it postgres pg_isready

# View logs
docker logs postgres
```

### Neo4j Memory Issues

```bash
# Increase heap size in docker-compose.yml
environment:
  - NEO4J_dbms_memory_heap_initial__size=1g
  - NEO4J_dbms_memory_heap_max__size=2g
```

### Elasticsearch Cluster Health

```bash
curl http://localhost:9200/_cluster/health?pretty
```
