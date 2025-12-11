# Troubleshooting Guide

## Common Issues and Solutions

### Service Startup Issues

#### Issue: Service fails to start with "Port already in use"

**Solution:**

```bash
# Windows - Find process using the port
netstat -ano | findstr :<port>

# Kill the process
taskkill /PID <pid> /F

# Or change the port in application.yml
server:
  port: 8081  # Change to unused port
```

#### Issue: "Connection refused" to database

**Solution:**

1. Verify Docker containers are running:

   ```bash
   docker-compose ps
   ```

2. Check database logs:

   ```bash
   docker logs postgres
   docker logs neo4j
   ```

3. Ensure network connectivity:

   ```bash
   docker network ls
   docker network inspect quickbooks-network
   ```

### Database Issues

#### Issue: PostgreSQL "FATAL: password authentication failed"

**Solution:**

```bash
# Reset password
docker exec -it postgres psql -U postgres -c "ALTER USER postgres PASSWORD 'postgres';"
```

#### Issue: Neo4j "Failed to obtain connection"

**Solution:**

1. Verify Neo4j is ready:

   ```bash
   docker exec -it neo4j cypher-shell -u neo4j -p password "RETURN 1"
   ```

2. Check memory settings - Neo4j needs at least 512MB heap

#### Issue: Elasticsearch "circuit_breaking_exception"

**Solution:**

```bash
# Increase JVM heap
docker exec -it elasticsearch bash -c "echo '-Xms2g' >> config/jvm.options"
docker-compose restart elasticsearch
```

### Kafka Issues

#### Issue: "Broker not available"

**Solution:**

1. Ensure Zookeeper is running first
2. Wait for Kafka to register with Zookeeper (30-60 seconds)
3. Check Kafka logs:

   ```bash
   docker logs kafka
   ```

#### Issue: Consumer lag increasing

**Solution:**

```bash
# Check consumer group status
docker exec -it kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe --group quickbooks-consumer
```

### Build Issues

#### Issue: Maven build fails with dependency errors

**Solution:**

```bash
# Clear local repository cache
mvn dependency:purge-local-repository

# Force update dependencies
mvn clean install -U -DskipTests
```

#### Issue: "OutOfMemoryError" during build

**Solution:**

```bash
# Increase Maven memory
set MAVEN_OPTS=-Xmx2048m -XX:MaxPermSize=512m
mvn clean install
```

### Frontend Issues

#### Issue: npm install fails

**Solution:**

```bash
# Clear npm cache
npm cache clean --force

# Delete node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
```

#### Issue: CORS errors in browser

**Solution:**

1. Verify Gateway CORS configuration in `application.yml`
2. Ensure API requests go through Gateway (port 8080)
3. Check browser developer tools for exact error

### Performance Issues

#### Issue: Slow API responses (>1s)

**Diagnosis:**

1. Check SLA metrics: `GET /api/v1/metrics/sla`
2. Review logs for slow queries
3. Check database connection pool utilization

**Solutions:**

- Enable caching for frequently accessed data
- Add database indexes for slow queries
- Scale horizontally with additional service instances

#### Issue: Memory leak in service

**Diagnosis:**

```bash
# Monitor JVM memory
jcmd <pid> VM.native_memory summary
```

**Solution:**

- Review code for unclosed resources
- Check for growing collections without bounds
- Enable heap dumps on OOM

### Entity Resolution Issues

#### Issue: Low match accuracy

**Solution:**

1. Review training data quality
2. Adjust confidence thresholds in config
3. Check standardization rules for edge cases

#### Issue: Entity Resolution service timeout

**Solution:**

1. Increase timeout in Gateway:

   ```yaml
   gateway:
     timeouts:
       entity-resolution: 60000  # 60 seconds
   ```

2. Enable async processing for large batches

### Logs and Monitoring

#### Enable Debug Logging

```yaml
logging:
  level:
    com.quickbooks: DEBUG
    org.springframework: INFO
    org.hibernate.SQL: DEBUG
```

#### Check Health Endpoints

```bash
# Check all services health
curl http://localhost:8080/actuator/health
curl http://localhost:8081/actuator/health
# ... repeat for each service
```

### Getting Help

1. Check service logs: `docker logs <service-name>`
2. Review application logs in `logs/` directory
3. Check Elasticsearch for indexed error events
4. Contact team via Slack #quickbooks-support
