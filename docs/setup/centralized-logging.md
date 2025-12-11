# Centralized Logging and Distributed Tracing

## Overview

The QuickBooks Business Network uses a centralized logging system with distributed tracing to provide unified visibility across all microservices. This eliminates the need to navigate through multiple terminal windows to diagnose issues.

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Java      │     │   Python    │     │   Other     │
│  Services   │     │  AI Service │     │  Services   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │ (Logback)         │ (Python)          │
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Kafka     │
                    │ application. │
                    │     logs     │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Logstash   │
                    │  (Parsing &  │
                    │  Enrichment) │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │Elasticsearch │
                    │  logs-*      │
                    │   indices    │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Kibana    │
                    │ (Dashboards) │
                    └──────────────┘
```

## Components

### 1. Spring Cloud Sleuth (Java Services)
- Automatically adds `trace_id` and `span_id` to all logs
- Propagates trace context across HTTP calls via B3 headers
- Integrates with Logback for structured JSON logging

### 2. Python Logging Integration (AI Service)
- Custom `KafkaLogHandler` sends logs to Kafka
- Extracts trace context from B3 headers (`X-B3-TraceId`, `X-B3-SpanId`)
- Formats logs as JSON with consistent structure

### 3. Kafka (Log Transport)
- Topic: `application.logs` (6 partitions)
- Asynchronous, non-blocking log delivery
- 7-day retention policy

### 4. Logstash (Log Processing)
- Consumes logs from Kafka
- Parses JSON and enriches with metadata
- Indexes to Elasticsearch with daily rotation (`logs-YYYY.MM.dd`)

### 5. Elasticsearch (Log Storage)
- Stores all logs with full-text search capability
- Daily index rotation for efficient management
- Retention managed via Index Lifecycle Management (ILM)

### 6. Kibana (Visualization)
- Pre-configured dashboards for service monitoring
- Log search with trace ID correlation
- Real-time log streaming

## Accessing Logs

### Kibana Web UI
1. Open browser to `http://localhost:5601`
2. Navigate to **Discover** to search logs
3. Use pre-configured dashboards:
   - **Service Overview**: Log volume, error rates, service health
   - **Error Analysis**: Error trends, stack traces, affected traces
   - **Request Tracing**: End-to-end request flows with timing

### Search by Trace ID
To trace a request across all services:
```
trace_id: "abc123def456"
```

### Search by Service
```
service_name: "business-service"
```

### Search for Errors
```
level: "ERROR"
```

### Combined Queries
```
service_name: "entity-resolution" AND level: "ERROR" AND message: *timeout*
```

## Log Format

All logs follow this JSON structure:

```json
{
  "timestamp": "2024-12-04T10:30:45.123Z",
  "level": "INFO",
  "service_name": "business-service",
  "logger_name": "com.quickbooks.business.service.BusinessService",
  "message": "Business entity created successfully",
  "trace_id": "abc123def456",
  "span_id": "789ghi012jkl",
  "thread": "http-nio-8081-exec-1",
  "context": {
    "business_id": "uuid-123",
    "user_id": "user-456"
  }
}
```

## Distributed Tracing

### How It Works
1. **Gateway** receives request and generates `trace_id`
2. **Sleuth** propagates `trace_id` via HTTP headers:
   - `X-B3-TraceId`: Unique trace identifier
   - `X-B3-SpanId`: Current span identifier
   - `X-B3-ParentSpanId`: Parent span identifier
3. Each service logs with the same `trace_id`
4. **Kibana** correlates logs by `trace_id`

### Example Trace Flow
```
Gateway (trace_id: abc123)
  ├─> Business Service (trace_id: abc123, span_id: span1)
  │     └─> Neo4j query
  └─> Entity Resolution (trace_id: abc123, span_id: span2)
        └─> AI Service (trace_id: abc123, span_id: span3)
              └─> ML model inference
```

## Configuration

### Java Services
Each service includes `logback-spring.xml` with:
- Console appender (human-readable for local dev)
- Kafka appender (JSON for centralized logging)
- Async wrapper to prevent blocking

Environment variables:
```bash
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
LOG_LEVEL=INFO
SPRING_APPLICATION_NAME=business-service
```

### Python AI Service
Environment variables:
```bash
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
LOG_LEVEL=INFO
ENABLE_KAFKA_LOGGING=true
```

## Troubleshooting

### Logs Not Appearing in Kibana

1. **Check Kafka topic**:
   ```bash
   docker exec -it qb-kafka kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic application.logs \
     --from-beginning --max-messages 10
   ```

2. **Check Logstash**:
   ```bash
   docker logs qb-logstash
   ```

3. **Check Elasticsearch indices**:
   ```bash
   curl http://localhost:9200/_cat/indices?v
   ```

4. **Verify Kibana index pattern**:
   - Go to Kibana > Stack Management > Index Patterns
   - Ensure `logs-*` pattern exists

### High Log Volume

If log volume is too high:
1. Increase log level to `WARN` or `ERROR`
2. Add filters in Logstash to drop verbose logs
3. Adjust Kafka retention policy

### Missing Trace IDs

If trace IDs are missing:
1. Verify Sleuth is enabled in `application.yml`:
   ```yaml
   spring:
     sleuth:
       enabled: true
       sampler:
         probability: 1.0  # Sample 100% of requests
   ```

2. Check B3 headers in HTTP requests:
   ```bash
   curl -H "X-B3-TraceId: test123" http://localhost:8080/api/v1/businesses
   ```

## Performance Considerations

- **Async Logging**: All Kafka appenders use async wrappers to prevent blocking
- **Batching**: Logstash batches messages before indexing
- **Sampling**: In production, consider sampling traces (e.g., 10%) to reduce volume
- **Retention**: Logs retained for 7 days by default

## Best Practices

1. **Always include context**: Add relevant business context to logs
   ```java
   logger.info("Business created", 
       Map.of("business_id", id, "category", category));
   ```

2. **Use appropriate log levels**:
   - `DEBUG`: Detailed diagnostic information
   - `INFO`: General informational messages
   - `WARN`: Warning messages for potentially harmful situations
   - `ERROR`: Error events that might still allow the application to continue

3. **Log exceptions with context**:
   ```java
   logger.error("Failed to create business", 
       Map.of("business_name", name), exception);
   ```

4. **Don't log sensitive data**: Never log passwords, tokens, or PII

5. **Use structured logging**: Always use key-value pairs for context

## Monitoring Alerts

Configure alerts in Kibana for:
- Error rate > 5% for any service
- No logs received from a service for > 5 minutes
- Specific error patterns (e.g., database connection failures)

## References

- Requirements: 51, 52, 53
- Spring Cloud Sleuth: https://spring.io/projects/spring-cloud-sleuth
- Logstash Configuration: `config/logstash/logstash.conf`
- Kibana Dashboards: `config/kibana/dashboards/`
