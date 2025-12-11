# End-to-End Testing Guide

This guide helps you validate that all components of the QuickBooks Business Network are working correctly from UI to backend services.

## Prerequisites

1. **Infrastructure Services Running** (via Docker Compose):
   ```powershell
   docker-compose ps
   ```
   All services should show "healthy" or "running":
   - PostgreSQL (port 5432)
   - Neo4j (port 7474, 7687)
   - Elasticsearch (port 9200)
   - Kafka (port 9092)
   - Redis (port 6379)
   - Milvus (port 19530)

2. **Microservices Running**:
   - API Gateway (port 8080)
   - Business Service (port 8081)
   - Network Service (port 8082)
   - Search Service (port 8083)
   - Transaction Service (port 8084)
   - Entity Resolution Service (port 8085)
   - Graph RAG Service (port 8086)
   - Ingest Flow Service (port 8087)
   - Resolve Flow Service (port 8088)

3. **AI Service Running** (port 8000):
   ```powershell
   curl http://localhost:8000/health
   ```
   Should return: `{"status":"healthy","model_loaded":true}`

4. **React UI Running** (port 3000):
   ```powershell
   curl http://localhost:3000
   ```

## Quick Start All Services

### Option 1: Use Startup Script (Recommended)
```powershell
.\scripts\start-complete.ps1
```

### Option 2: Manual Start
```powershell
# Start infrastructure
docker-compose up -d

# Start Java services (in separate terminals or background)
cd src/services/gateway && mvn spring-boot:run
cd src/services/business && mvn spring-boot:run
cd src/services/network && mvn spring-boot:run
cd src/services/search && mvn spring-boot:run
cd src/services/transaction && mvn spring-boot:run
cd src/services/entity-resolution && mvn spring-boot:run
cd src/services/graph-rag && mvn spring-boot:run
cd src/services/ingest-flow && mvn spring-boot:run
cd src/services/resolve-flow && mvn spring-boot:run

# Start AI Service
cd src/ai-service
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000

# Start Frontend
cd src/frontend/react-ui
npm run dev
```

## Testing Checklist

### 1. Test Infrastructure Services

```powershell
# PostgreSQL
docker exec qb-postgres pg_isready -U postgres

# Neo4j
curl http://localhost:7474

# Elasticsearch
curl http://localhost:9200/_cluster/health

# Kafka
curl http://localhost:9000

# Redis
docker exec qb-redis redis-cli ping
```

### 2. Test API Gateway

```powershell
# Health check
curl http://localhost:8080/actuator/health

# Should return: {"status":"UP"}
```

### 3. Test Business Service

```powershell
# List businesses
curl "http://localhost:8080/api/v1/businesses?page=0&size=5"

# Should return paginated list of businesses
```

### 4. Test Network Service & Data Calculation

```powershell
# Get a business ID first
$businessId = (curl "http://localhost:8080/api/v1/businesses?page=0&size=1" | ConvertFrom-Json).content[0].id

# Get network map (includes weight calculation)
curl "http://localhost:8080/api/v1/networks/$businessId/map?depth=1&include_weights=true"

# Verify response includes:
# - nodes[] array
# - edges[] array with weight property
```

### 5. Test Search Service

```powershell
# Search entities
curl "http://localhost:8080/api/v1/search/entities?q=test&size=5"

# Should return search results with relevance scores

# Get connected nodes
$entityId = (curl "http://localhost:8080/api/v1/search/entities?q=test&size=1" | ConvertFrom-Json).results[0].id
curl "http://localhost:8080/api/v1/search/entities/$entityId/connected?depth=2"

# Should return connected nodes and edges from Neo4j
```

### 6. Test AI Service (Entity Resolution)

```powershell
# Health check
curl http://localhost:8000/health

# Test classification
$body = @{
    feature_vector = @{
        levenshtein_score = 0.85
        jaro_winkler_score = 0.90
        token_similarity = 0.88
        address_similarity = 0.75
        category_match = 1.0
        same_legal_type = 1.0
        same_state = 1.0
    }
    entity_a_id = "test-a"
    entity_b_id = "test-b"
} | ConvertTo-Json

curl -Method POST -Uri "http://localhost:8000/api/v1/classify" -Body $body -ContentType "application/json"

# Should return:
# {
#   "match_probability": 0.92,
#   "confidence": 0.88,
#   "processing_time_ms": 15
# }
```

### 7. Test Transaction Weight Calculation

```powershell
# Get relationship weight
$relationshipId = "some-relationship-id"
curl "http://localhost:8080/api/v1/transactions/relationships/$relationshipId/weight"

# Should return:
# {
#   "transaction_volume": 10000.00,
#   "weight": 0.01,
#   "transaction_count": 5
# }
```

### 8. Test UI

1. **Open Browser**: Navigate to `http://localhost:3000`

2. **Test Home Page**:
   - Should display list of businesses
   - Click on a business to select it
   - Should navigate to dashboard

3. **Test Dashboard**:
   - **Network Graph Tab**: Should display D3.js force-directed graph
   - **Search Tab**: Should allow searching for businesses
   - **Relationships Tab**: Should show relationship management
   - **Chat Tab**: Should allow Graph RAG queries

4. **Test Button Clicks**:
   - Click "Add Relationship" button
   - Click "Delete Relationship" button (with confirmation)
   - Click search button
   - Click export button
   - Click feedback button

5. **Test Network Graph**:
   - Graph should render nodes and edges
   - Click on nodes to see details
   - Weights should be displayed on edges
   - Zoom and pan should work

6. **Test Search**:
   - Type in search box
   - Results should appear with relevance scores
   - Click on result to see connected nodes

7. **Test Graph RAG Chat**:
   - Type natural language query: "flower businesses in Texas connected to me"
   - Should return results with explanations

## Expected Results

### Network Data Calculation
- Transaction volumes are summed correctly
- Relationship weights are calculated (0.0-1.0)
- Weights are displayed in network graph
- Weight updates when transactions are added

### Search Functionality
- Elasticsearch queries return results quickly (<500ms)
- Results are ranked by relevance
- Connected nodes are retrieved from Neo4j
- Graph traversal works up to 3 levels deep

### AI Service
- Classification endpoint responds (<100ms)
- Match probability is between 0.0-1.0
- Confidence score is provided
- Model is loaded and ready

### UI Interactions
- All buttons respond to clicks
- Forms validate input
- API calls succeed
- Error messages display correctly
- Loading states show during API calls

## Troubleshooting

### Services Not Starting
1. Check logs: `logs/*.log`
2. Verify ports are not in use: `netstat -ano | findstr "8080 8081 8082"`
3. Check Docker containers: `docker ps`
4. Verify environment variables are set

### API Calls Failing
1. Check API Gateway is running: `curl http://localhost:8080/actuator/health`
2. Check service logs for errors
3. Verify database connections
4. Check CORS settings

### UI Not Loading
1. Check frontend is running: `curl http://localhost:3000`
2. Check browser console for errors
3. Verify API base URL in `.env` file
4. Check network tab for failed requests

### Search Not Working
1. Verify Elasticsearch is healthy: `curl http://localhost:9200/_cluster/health`
2. Check if index exists: `curl http://localhost:9200/business_entities`
3. Verify data is indexed
4. Check Search Service logs

### AI Service Not Responding
1. Check AI service health: `curl http://localhost:8000/health`
2. Verify model is loaded: `model_loaded: true`
3. Check Python dependencies: `pip list`
4. Review AI service logs

## Performance Targets

- Network Map Load: < 2 seconds (95th percentile)
- Entity Search: < 500ms (95th percentile)
- Graph Traversal: < 1 second (95th percentile)
- Entity Resolution: < 100ms (95th percentile)
- AI Classification: < 100ms (95th percentile)

## Next Steps

After validating all components:

1. Seed test data: `.\scripts\seed-data.ps1`
2. Run integration tests: `mvn test`
3. Check metrics: `http://localhost:8080/actuator/metrics`
4. Review logs in Kibana: `http://localhost:5601`

