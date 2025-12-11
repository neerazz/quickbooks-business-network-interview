# Operations Runbooks

## Overview

This document contains runbooks for common operational tasks and incident response.

## Service Health Checks

### Check All Services

```bash
#!/bin/bash
SERVICES=("gateway:8080" "business:8081" "network:8082" "search:8083" "transaction:8084" "entity-resolution:8085" "graph-rag:8086")

for service in "${SERVICES[@]}"; do
    name="${service%%:*}"
    port="${service##*:}"
    status=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$port/actuator/health)
    if [ "$status" == "200" ]; then
        echo "[OK] $name ($port) is healthy"
    else
        echo "[FAIL] $name ($port) is unhealthy (HTTP $status)"
    fi
done
```

### Check Database Connections

```bash
# PostgreSQL
docker exec -it postgres pg_isready -U postgres

# Neo4j
docker exec -it neo4j cypher-shell -u neo4j -p password "RETURN 1"

# Elasticsearch
curl -s http://localhost:9200/_cluster/health | jq '.status'

# Redis
docker exec -it redis redis-cli PING

# Kafka
docker exec -it kafka kafka-broker-api-versions.sh --bootstrap-server localhost:9092 | head -1
```

## Incident Response

### Runbook: High API Latency

**Symptoms:**

- P95 response time > 1000ms
- User complaints about slow responses

**Steps:**

1. Check SLA metrics dashboard

   ```bash
   curl http://localhost:8091/api/v1/metrics/sla
   ```

2. Identify slow endpoints

   ```bash
   curl http://localhost:8091/api/v1/metrics/endpoints | jq 'sort_by(.p95) | reverse | .[0:5]'
   ```

3. Check database performance

   ```bash
   # PostgreSQL slow queries
   docker exec -it postgres psql -U postgres -c "SELECT query, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 5;"
   
   # Neo4j query log
   docker exec -it neo4j tail -100 /logs/query.log
   ```

4. Check service resource usage

   ```bash
   docker stats --no-stream
   ```

5. Scale if needed

   ```bash
   docker-compose up -d --scale network-service=3
   ```

### Runbook: Entity Resolution Backlog

**Symptoms:**

- Kafka consumer lag increasing
- Pending resolutions not being processed

**Steps:**

1. Check Kafka consumer lag

   ```bash
   docker exec -it kafka kafka-consumer-groups.sh \
     --bootstrap-server localhost:9092 \
     --describe --group entity-resolution-consumer
   ```

2. Check Entity Resolution service logs

   ```bash
   docker logs --tail 100 entity-resolution-service
   ```

3. Verify AI service is responding

   ```bash
   curl http://localhost:8000/health
   ```

4. If backlog, increase consumers

   ```bash
   docker-compose up -d --scale entity-resolution=3
   ```

5. Clear stuck messages (if needed)

   ```bash
   docker exec -it kafka kafka-console-consumer.sh \
     --bootstrap-server localhost:9092 \
     --topic entity.pending \
     --from-beginning --max-messages 10
   ```

### Runbook: Database Connection Exhaustion

**Symptoms:**

- "Unable to acquire connection" errors
- Services timing out on database operations

**Steps:**

1. Check connection pool status

   ```bash
   curl http://localhost:8081/actuator/metrics/hikaricp.connections.active
   curl http://localhost:8081/actuator/metrics/hikaricp.connections.pending
   ```

2. Identify connection leaks

   ```bash
   docker exec -it postgres psql -U postgres -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
   ```

3. Kill idle connections

   ```bash
   docker exec -it postgres psql -U postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND query_start < now() - interval '10 minutes';"
   ```

4. Increase pool size (if needed)
   - Update `application.yml`:

     ```yaml
     spring:
       datasource:
         hikari:
           maximum-pool-size: 30
     ```

   - Restart affected services

## Backup Procedures

### PostgreSQL Backup

```bash
# Full backup
docker exec -t postgres pg_dumpall -U postgres > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore
cat backup.sql | docker exec -i postgres psql -U postgres
```

### Neo4j Backup

```bash
# Stop Neo4j for consistent backup
docker-compose stop neo4j

# Backup data volume
docker run --rm -v neo4j_data:/data -v $(pwd):/backup alpine \
  tar cvf /backup/neo4j_backup_$(date +%Y%m%d).tar /data

# Start Neo4j
docker-compose start neo4j
```

### Elasticsearch Backup

```bash
# Create snapshot repository
curl -X PUT "localhost:9200/_snapshot/backup" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": { "location": "/backup" }
}'

# Create snapshot
curl -X PUT "localhost:9200/_snapshot/backup/snapshot_$(date +%Y%m%d)"
```

## Scaling Procedures

### Horizontal Scaling

```bash
# Scale specific service
docker-compose up -d --scale business-service=3

# Verify instances
docker ps | grep business-service
```

### Vertical Scaling

Update resource limits in `docker-compose.yml`:

```yaml
services:
  business-service:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
```

## Maintenance Windows

### Planned Maintenance Checklist

1. [ ] Notify users 48 hours in advance
2. [ ] Enable maintenance mode in Gateway
3. [ ] Stop accepting new requests
4. [ ] Drain existing connections
5. [ ] Perform maintenance
6. [ ] Verify health checks pass
7. [ ] Disable maintenance mode
8. [ ] Monitor for 30 minutes
9. [ ] Update status page

### Rolling Restart

```bash
# Restart services one at a time
SERVICES=("gateway:8080" "business:8081" "network:8082" "search:8083" "transaction:8084" "entity-resolution:8085" "graph-rag:8086")

for service in "${SERVICES[@]}"; do
    name="${service%%:*}"
    port="${service##*:}"

    echo "Restarting $name..."
    docker-compose restart "$name"
    sleep 30  # Wait for health check
    
    health=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$port/actuator/health)
    if [ "$health" != "200" ]; then
        echo "ERROR: $name failed health check!"
        exit 1
    fi
done
```

## Monitoring Alerts

### Alert Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| P95 Latency | > 500ms | > 1000ms |
| Error Rate | > 1% | > 5% |
| CPU Usage | > 70% | > 90% |
| Memory Usage | > 75% | > 90% |
| Kafka Lag | > 1000 | > 10000 |
| DB Connections | > 80% | > 95% |

### Alert Response

When receiving an alert:

1. Acknowledge alert in monitoring system
2. Open incident ticket
3. Follow relevant runbook
4. Update incident timeline
5. Resolve and document RCA
