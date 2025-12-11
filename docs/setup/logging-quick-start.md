# Centralized Logging - Quick Start Guide

## Getting Started in 5 Minutes

### 1. Start the Infrastructure

```bash
# Start all services including ELK stack
docker-compose up -d

# Wait for services to be healthy (2-3 minutes)
docker-compose ps
```

### 2. Verify Logging Infrastructure

```bash
# Check Elasticsearch
curl http://localhost:9200/_cluster/health

# Check Logstash
curl http://localhost:9600/_node/stats

# Check Kibana
curl http://localhost:5601/api/status
```

### 3. Access Kibana Dashboard

1. Open browser: `http://localhost:5601`
2. Wait for Kibana to initialize (30-60 seconds)
3. Go to **Discover** in the left sidebar

### 4. Create Index Pattern (First Time Only)

1. In Kibana, go to **Stack Management** > **Index Patterns**
2. Click **Create index pattern**
3. Enter pattern: `logs-*`
4. Select time field: `@timestamp`
5. Click **Create index pattern**

### 5. View Logs

1. Go to **Discover** in the left sidebar
2. Select `logs-*` index pattern
3. You should see logs from all services!

## Common Search Queries

### Find All Errors
```
level: "ERROR"
```

### Find Logs from Specific Service
```
service_name: "business-service"
```

### Trace a Request Across Services
```
trace_id: "abc123def456"
```

### Find Slow Requests
```
message: *timeout* OR message: *slow*
```

### Find Database Errors
```
message: *database* AND level: "ERROR"
```

## Pre-Configured Dashboards

Access these dashboards from Kibana's **Dashboard** section:

1. **Service Overview**
   - Log volume by service
   - Error rates
   - Service health indicators

2. **Error Analysis**
   - Error trends over time
   - Top error messages
   - Stack traces

3. **Request Tracing**
   - End-to-end request flows
   - Service call graphs
   - Response time distribution

## Debugging a Request

### Scenario: User reports an error creating a business

1. **Find the error in Kibana**:
   ```
   service_name: "business-service" AND level: "ERROR" AND message: *create*
   ```

2. **Get the trace ID** from the log entry

3. **Trace the full request**:
   ```
   trace_id: "the-trace-id-from-step-2"
   ```

4. **View the timeline**:
   - Click on any log entry
   - Click **View surrounding documents**
   - See the full request flow across all services

### Example Trace Flow

```
14:30:45.123 | gateway          | INFO  | Request received: POST /api/v1/businesses
14:30:45.145 | business-service | INFO  | Creating business: ACME Corp
14:30:45.167 | business-service | DEBUG | Validating business name
14:30:45.189 | business-service | DEBUG | Checking for duplicates in Neo4j
14:30:45.234 | business-service | ERROR | Neo4j connection timeout
14:30:45.256 | gateway          | ERROR | Request failed: 500 Internal Server Error
```

## Troubleshooting

### No Logs Appearing?

1. **Check if services are logging to Kafka**:
   ```bash
   docker exec -it qb-kafka kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic application.logs \
     --from-beginning --max-messages 5
   ```

2. **Check Logstash is consuming**:
   ```bash
   docker logs qb-logstash | grep "Pipeline started"
   ```

3. **Check Elasticsearch indices**:
   ```bash
   curl http://localhost:9200/_cat/indices?v | grep logs
   ```

### Logs Are Delayed?

- Logstash batches logs for efficiency
- Default batch delay: 5 seconds
- Check Logstash config: `config/logstash/logstash.conf`

### Kibana Won't Load?

1. **Check Elasticsearch is healthy**:
   ```bash
   curl http://localhost:9200/_cluster/health
   ```

2. **Restart Kibana**:
   ```bash
   docker-compose restart kibana
   ```

3. **Check Kibana logs**:
   ```bash
   docker logs qb-kibana
   ```

## Performance Tips

### For Development
- Keep all logging enabled
- Sample 100% of traces
- Use `LOG_LEVEL=DEBUG` for your service

### For Production
- Set `LOG_LEVEL=INFO` or `WARN`
- Sample 10% of traces: `SPRING_SLEUTH_SAMPLER_PROBABILITY=0.1`
- Filter verbose logs in Logstash

## Best Practices

1. **Always include context**:
   ```java
   logger.info("Business created", 
       Map.of("business_id", id, "user_id", userId));
   ```

2. **Use trace IDs for debugging**:
   - Copy trace ID from error logs
   - Search all services with that trace ID
   - See the complete request flow

3. **Set up alerts**:
   - Error rate > 5%
   - Service not logging for > 5 minutes
   - Specific error patterns

4. **Regular cleanup**:
   - Logs are retained for 7 days
   - Old indices are automatically deleted
   - Adjust retention in `scripts/db/init-kafka.sh`

## Next Steps

- Read full documentation: `docs/setup/centralized-logging.md`
- Configure custom dashboards in Kibana
- Set up alerting for critical errors
- Integrate with your monitoring tools

## Need Help?

- Check service logs: `docker logs <container-name>`
- View all containers: `docker-compose ps`
- Restart a service: `docker-compose restart <service-name>`
- Full reset: `docker-compose down -v && docker-compose up -d`

---

**Happy debugging!**

No more navigating through multiple terminals - all your logs are now in one place!
