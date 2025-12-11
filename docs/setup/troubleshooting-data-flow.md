# Troubleshooting Data Flow Issues

This guide helps diagnose and resolve common data flow problems in the QuickBooks Business Network.

## Quick Diagnostics

Run the E2E test script for automated diagnostics:

```powershell
# Windows
.\scripts\test-e2e-flow.ps1

# Linux/Mac
./scripts/test-e2e-flow.sh
```

## Common Issues

### 1. Events Stuck in PENDING Status

**Symptoms:**
- Ingest metrics show high `pendingEvents`
- Events don't appear in Kafka

**Diagnosis:**
```sql
-- Check PostgreSQL for pending events
docker exec qb-postgres psql -U postgres -d quickbooks_network -c \
  "SELECT COUNT(*), status FROM quickbooks_events GROUP BY status;"
```

**Solutions:**
- Run the process-pending endpoint:
  ```bash
  curl -X POST "http://localhost:8087/api/v1/ingest/process-pending?batchSize=1000"
  ```
- Check Kafka connectivity:
  ```bash
  docker exec qb-kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
  ```

---

### 2. Kafka Consumer Lag

**Symptoms:**
- Events published but not processing
- Neo4j/Elasticsearch counts not increasing

**Diagnosis:**
```bash
# Check consumer group lag
docker exec qb-kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group resolve-flow-group \
  --describe
```

**Solutions:**
- Check Resolve Flow service is running
- Review logs: `logs/resolve-flow.log`
- Restart the service if needed

---

### 3. Neo4j Sync Issues

**Symptoms:**
- Events processed but nodes not created
- Relationship creation fails

**Diagnosis:**
```cypher
-- Check Neo4j node count
docker exec qb-neo4j cypher-shell -u neo4j -p neo4j_password \
  "MATCH (n:Business) RETURN COUNT(n);"
```

**Solutions:**
- Check Neo4j connection in Resolve Flow config
- Verify Neo4j is accepting connections:
  ```bash
  curl http://localhost:7474
  ```
- Review entity resolution logs for failures

---

### 4. Elasticsearch Indexing Delays

**Symptoms:**
- Nodes exist in Neo4j but not searchable
- Search returns no results

**Diagnosis:**
```bash
# Check Elasticsearch index
curl "http://localhost:9200/business_entities/_count"

# Check cluster health
curl "http://localhost:9200/_cluster/health"
```

**Solutions:**
- Wait for index refresh (default 5 seconds)
- Force refresh:
  ```bash
  curl -X POST "http://localhost:9200/business_entities/_refresh"
  ```
- Check Elasticsearch connection in Search service

---

### 5. Entity Resolution Failures

**Symptoms:**
- Events processed but entities not resolved
- Duplicate entities created

**Diagnosis:**
- Check AI service health: `http://localhost:8095/health`
- Review entity resolution logs

**Solutions:**
- Ensure AI service is running
- Check embedding model is loaded
- Verify vector DB connectivity

## Service Health Checks

| Service | Health Endpoint | Port |
|---------|-----------------|------|
| Gateway | `/actuator/health` | 8080 |
| Ingest Flow | `/actuator/health` | 8087 |
| Resolve Flow | `/actuator/health` | 8088 |
| Search | `/actuator/health` | 8083 |
| Entity Resolution | `/actuator/health` | 8085 |

## Log Locations

- Ingest Flow: `logs/ingest_flow_run.log`
- Resolve Flow: `logs/resolve_flow_run.log`
- Search: `logs/search_run.log`
- AI Service: `logs/ai_service.log`

## Metrics Endpoints

```bash
# Ingest metrics
curl http://localhost:8087/api/v1/ingest/metrics

# Resolve metrics
curl http://localhost:8088/api/v1/resolve/metrics

# Search metrics
curl http://localhost:8083/api/v1/search/metrics
```

## Complete Reset

If issues persist, reset the entire data pipeline:

```powershell
# Stop all services
docker-compose down -v

# Restart fresh
docker-compose up -d
.\scripts\start.ps1
```

> **Warning:** This will delete all data. Use only in development.
