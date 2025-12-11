# End-to-End Validation Status

## Current Status

### ✅ Infrastructure Services (Running)
All Docker Compose services are healthy:
- PostgreSQL (port 5432) ✓
- Neo4j (port 7474, 7687) ✓
- Elasticsearch (port 9200) ✓
- Kafka (port 9092) ✓
- Redis (port 6379) ✓
- Milvus (port 19530) ✓
- Zookeeper (port 2181) ✓

### ✅ AI Service (Running)
- Port: 8000
- Status: Healthy
- Model: Loaded
- Test: `curl http://localhost:8000/health` returns `{"status":"healthy","model_loaded":true}`

### ⚠️ Microservices (Need to Start)
The following Java microservices need to be started:
- API Gateway (port 8080)
- Business Service (port 8081)
- Network Service (port 8082)
- Search Service (port 8083)
- Transaction Service (port 8084)
- Entity Resolution Service (port 8085)
- Graph RAG Service (port 8086)
- Ingest Flow Service (port 8087)
- Resolve Flow Service (port 8088)

### ⚠️ React UI (Need to Start)
- Port: 3000
- Status: Not running

## Quick Start Commands

### Start All Services
```powershell
# Option 1: Use startup script
.\scripts\start-complete.ps1

# Option 2: Manual start (see below)
```

### Manual Start

#### 1. Start Infrastructure (Already Running)
```powershell
docker-compose up -d
```

#### 2. Start Java Microservices
```powershell
# In separate terminals or as background processes:
cd src/services/gateway && mvn spring-boot:run
cd src/services/business && mvn spring-boot:run
cd src/services/network && mvn spring-boot:run
cd src/services/search && mvn spring-boot:run
cd src/services/transaction && mvn spring-boot:run
cd src/services/entity-resolution && mvn spring-boot:run
cd src/services/graph-rag && mvn spring-boot:run
cd src/services/ingest-flow && mvn spring-boot:run
cd src/services/resolve-flow && mvn spring-boot:run
```

#### 3. Start AI Service (Already Running)
```powershell
cd src/ai-service
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

#### 4. Start React UI
```powershell
cd src/frontend/react-ui
npm install  # If not already done
npm run dev
```

## Testing Checklist

Once all services are running, test the following:

### 1. Test API Gateway
```powershell
curl http://localhost:8080/actuator/health
# Should return: {"status":"UP"}
```

### 2. Test Business Service
```powershell
curl "http://localhost:8080/api/v1/businesses?page=0&size=5"
# Should return paginated list of businesses
```

### 3. Test Network Data Calculation
```powershell
# Get a business ID
$businessId = (Invoke-WebRequest -Uri "http://localhost:8080/api/v1/businesses?page=0&size=1" | ConvertFrom-Json).content[0].id

# Get network map with weights
Invoke-WebRequest -Uri "http://localhost:8080/api/v1/networks/$businessId/map?depth=1&include_weights=true" | ConvertFrom-Json
# Should return nodes and edges with weight property
```

### 4. Test Search Service
```powershell
# Search entities
Invoke-WebRequest -Uri "http://localhost:8080/api/v1/search/entities?q=test&size=5" | ConvertFrom-Json
# Should return search results with relevance scores
```

### 5. Test AI Service Classification
```powershell
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
} | ConvertTo-Json -Depth 10

Invoke-WebRequest -Uri "http://localhost:8000/api/v1/classify" -Method POST -Body $body -ContentType "application/json" | ConvertFrom-Json
# Should return: match_probability, confidence, processing_time_ms
```

### 6. Test UI
1. Open browser: `http://localhost:3000`
2. Verify:
   - Home page loads with business list
   - Click business to navigate to dashboard
   - Network graph displays
   - Search works
   - Buttons respond to clicks
   - Forms validate input

## Component Status Summary

| Component | Status | Port | Notes |
|-----------|--------|------|-------|
| Infrastructure | ✅ Running | Various | All Docker services healthy |
| AI Service | ✅ Running | 8000 | Model loaded, ready for classification |
| API Gateway | ⚠️ Not Running | 8080 | Need to start |
| Business Service | ⚠️ Not Running | 8081 | Need to start |
| Network Service | ⚠️ Not Running | 8082 | Need to start |
| Search Service | ⚠️ Not Running | 8083 | Need to start |
| Transaction Service | ⚠️ Not Running | 8084 | Need to start |
| Entity Resolution | ⚠️ Not Running | 8085 | Need to start |
| Graph RAG | ⚠️ Not Running | 8086 | Need to start |
| Ingest Flow | ⚠️ Not Running | 8087 | Need to start |
| Resolve Flow | ⚠️ Not Running | 8088 | Need to start |
| React UI | ⚠️ Not Running | 3000 | Need to start |

## Next Steps

1. **Start all microservices** using the commands above
2. **Start the React UI**
3. **Seed test data** (optional): `.\scripts\seed-data.ps1`
4. **Run validation tests** as described in `E2E-TESTING-GUIDE.md`
5. **Verify end-to-end flow**:
   - UI loads correctly
   - Button clicks work
   - Network data calculation works
   - Search functionality works
   - AI service classification works

## Documentation

- **Full Testing Guide**: `docs/validation/E2E-TESTING-GUIDE.md`
- **Setup Guide**: `docs/setup/getting-started.md`
- **Architecture**: `docs/architecture/overview.md`

## Troubleshooting

If services fail to start:
1. Check logs: `logs/*.log`
2. Verify ports are not in use: `netstat -ano | findstr "8080 8081 8082"`
3. Check Docker containers: `docker ps`
4. Verify environment variables are set
5. Check service-specific logs in `src/services/*/logs/`

