# QuickBooks Business Network - Quick Reference Guide

## Two Business Creation Workflows

---

## Workflow 1: Manual Business Creation (User-Initiated)

### Trigger
User explicitly creates a business under their account via UI

### Flow Summary
```
UI → Business Service → PostgreSQL → Kafka → Network Service → Entity Resolution → Neo4j → Kafka → Search Service
```

### Detailed Steps

1. **User Action**
   - User fills form: "Create new business"
   - Input: name, address, category, etc.

2. **Business Service** (Port 8081)
   - Validates business data
   - INSERT into PostgreSQL → `business_id = 12345`
   - PUBLISH to Kafka: `business.events`

3. **Network Service** (Port 8082) - **ORCHESTRATOR**
   - Consumes event from Kafka (consumer group: network-service)
   - Triggers Entity Resolution via HTTP

4. **Entity Resolution** (Port 8085) - **4-STAGE PIPELINE**
   - **Stage 1**: AI Service → Standardize name
     ```
     "Acme Corp" → "ACME CORPORATION"
     ```
   
   - **Stage 2**: Search Service → Fuzzy matching
     ```
     HTTP GET /search/fuzzy?q=ACME CORPORATION&maxResults=10
     → Elasticsearch: n-gram + phonetic + fuzzy queries
     → Returns: Top 10 candidates with scores
     ```
   
   - **Stage 3**: Vector DB → Semantic similarity
     ```
     Generate embedding → Query Milvus → Rank candidates
     ```
   
   - **Stage 4**: AI Service → ML Classification
     ```
     XGBoost model → {is_duplicate: true/false, confidence: 0.92}
     ```

5. **Network Service** - **DECISION POINT**
   
   **IF Duplicate Found** (confidence > 0.85):
   ```
   Neo4j: UPDATE existing node
   Link duplicate_of: [12345]
   NO new node created
   NO indexing needed
   ```
   
   **IF New Business** (confidence < 0.85):
   ```
   Neo4j: CREATE (Business:12345 {name, address, ...})
   Kafka: PUBLISH to search.index.requests
   Redis: Invalidate cache
   ```

6. **Search Service** (Port 8083)
   - Consumes from Kafka (consumer group: search-service)
   - INDEX new business in Elasticsearch
   - Warm up Redis cache

### Latency
- Business Service → PostgreSQL: 10ms
- Kafka publish: 5ms
- Entity Resolution (4 stages): 75ms
- Neo4j operations: 20ms
- Search indexing (async): 5ms
- **Total**: ~175ms

---

## Workflow 2: Transaction-Based Auto Node Creation

### Trigger
User records a transaction with an external business that doesn't exist in the system

### Flow Summary
```
UI → Transaction Service → Payment Gateway → PostgreSQL → Kafka → Network Service → Entity Resolution → Neo4j (create + relationship) → Kafka → Search Service
```

### Detailed Steps

1. **User Action**
   - User records transaction: "Paid $15,000 to New Supplier Corp"
   - External business not in system

2. **Transaction Service** (Port 8084)
   - Process payment via Payment Gateway
   - Wait for approval ✓
   - INSERT into PostgreSQL:
     ```sql
     INSERT INTO transactions (
       from_business_id: 12345,
       to_business_name: "New Supplier Corp",
       amount: 15000,
       status: "approved"
     )
     ```
   - PUBLISH to Kafka: `transaction.events` (ONLY after approval)

3. **Network Service** (Port 8082) - **ORCHESTRATOR**
   - Consumes event from Kafka
   - Check Neo4j: Does "New Supplier Corp" exist?
     ```cypher
     MATCH (b:Business {name: "New Supplier Corp"}) RETURN b
     ```
   - Result: NOT FOUND → Trigger auto-creation

4. **Entity Resolution** (Port 8085) - **SAME 4-STAGE PIPELINE**
   - Stage 1: AI standardize
   - Stage 2: Search Service fuzzy search
   - Stage 3: Vector DB semantic similarity
   - Stage 4: AI ML prediction
   - Result: `{is_duplicate: false, confidence: 0.10}`

5. **Network Service** - **AUTO-CREATE NODE**
   ```cypher
   // Create new business node
   CREATE (b:Business {
     id: 67890,  // auto-generated
     name: "New Supplier Corp",
     auto_created: true,
     source: "transaction",
     created_from_txn: 9999
   })
   
   // Create relationship immediately
   CREATE (from:Business {id: 12345})
         -[r:TRANSACTS_WITH {
           amount: 15000,
           date: "2025-12-11",
           count: 1
         }]->(to:Business {id: 67890})
   ```
   - PUBLISH to Kafka: `search.index.requests`

6. **Search Service** (Port 8083)
   - INDEX new auto-created business in Elasticsearch

7. **Payment Service** (Port 8084) - **PARALLEL**
   - Processes payment settlement
   - Updates transaction status

### Latency
- Transaction Service → Payment: 125ms
- PostgreSQL insert: 10ms
- Kafka publish: 5ms
- Entity Resolution: 75ms
- Neo4j create + relationship: 35ms
- **Total**: ~250ms

---

## Key Architectural Components

### Network Service (Port 8082) - ORCHESTRATOR
**Role**: Coordinates all graph operations and entity resolution

**Consumes**:
- `business.events` (manual creation)
- `transaction.events` (auto-creation)

**Produces**:
- `search.index.requests` (when new nodes created)

**Calls**:
- Entity Resolution (HTTP) → Trigger 4-stage pipeline
- Neo4j → Create/update nodes and relationships

---

### Entity Resolution Service (Port 8085) - STATELESS ML ENGINE
**Role**: Determine if business is duplicate or new

**4-Stage Pipeline**:
1. AI Service (HTTP) → Name standardization
2. **Search Service (HTTP)** → Fuzzy matching ← **CRITICAL CONNECTION**
3. Vector DB (gRPC) → Semantic similarity
4. AI Service (HTTP) → ML classification

**Does NOT**:
- ❌ Consume from Kafka
- ❌ Access Elasticsearch directly
- ❌ Create Neo4j nodes
- ❌ Index in Elasticsearch

**Returns to Network Service**:
```json
{
  "is_duplicate": true/false,
  "matched_business_id": 54321,
  "confidence": 0.92,
  "reason": "High similarity + ML confidence"
}
```

---

### Search Service (Port 8083) - DUAL ROLE
**Role 1**: User-facing search API
**Role 2**: Fuzzy matching for Entity Resolution

**Consumes**:
- `search.index.requests` (from Network Service)

**Provides**:
- GET `/search` → User search queries
- POST `/search/fuzzy` → Entity Resolution blocking ← **CALLED BY ENTITY RESOLUTION**

**Fuzzy Search Strategies**:
- N-gram matching (partial text)
- Phonetic matching (sound-alike)
- Fuzzy matching (typo tolerance)
- Geographic filtering (nearby businesses)

---

## Critical Connections

### Network Service → Entity Resolution
```
HTTP POST http://entity-resolution:8085/resolve
{
  "businessId": 12345,
  "name": "Acme Corp",
  "address": {...}
}
```

### Entity Resolution → Search Service
```
HTTP POST http://search-service:8083/search/fuzzy
{
  "query": "ACME CORPORATION",
  "maxResults": 10,
  "includePhonetic": true,
  "includeNgram": true,
  "minScore": 5.0
}
```

### Network Service → Kafka
```
Topic: search.index.requests
{
  "event": "IndexBusiness",
  "business_id": 67890,
  "action": "create",
  "source": "transaction_auto_create"
}
```

---

## Why Entity Resolution Calls Search Service (Not Elasticsearch Directly)

### Benefits
1. **Encapsulation**: Search Service owns Elasticsearch schema and queries
2. **Reusability**: Same fuzzy logic for user search + entity matching
3. **Caching**: Search Service caches fuzzy results
4. **Evolution**: Easy to enhance matching without changing Entity Resolution
5. **Separation of Concerns**: Entity Resolution focuses on ML, Search Service on queries

### Alternative Rejected
❌ Entity Resolution → Elasticsearch directly
- Duplicates search query logic
- Tight coupling to Elasticsearch
- No caching benefits
- Hard to evolve independently

---

## Comparison: Manual vs Transaction-Based Creation

| Aspect | Manual Creation | Transaction Auto-Creation |
|--------|----------------|---------------------------|
| **Trigger** | User form submission | Transaction approval |
| **Entry Point** | Business Service | Transaction Service |
| **Data Source** | User input | Transaction data |
| **Relationship** | Optional | Immediate (TRANSACTS_WITH) |
| **Node Flag** | `auto_created: false` | `auto_created: true` |
| **Source** | `source: "user"` | `source: "transaction"` |
| **Latency** | ~175ms | ~250ms (includes payment) |
| **Use Case** | Onboarding, explicit setup | Recording external transactions |

---

## Service Ports Quick Reference

| Service | Port | Role |
|---------|------|------|
| React UI | 3000 | Frontend |
| API Gateway | 8080 | Entry point |
| Business Service | 8081 | Business CRUD + Kafka producer |
| Network Service | 8082 | **ORCHESTRATOR** + Neo4j |
| Search Service | 8083 | **Search + Fuzzy matching** |
| Transaction Service | 8084 | Transaction recording |
| Entity Resolution | 8085 | **ML-powered matching** |
| AI Service | 8000 | Standardization + ML + Embeddings |
| PostgreSQL | 5432 | Source of truth |
| Neo4j | 7687 | Graph database |
| Elasticsearch | 9200 | Search engine |
| Kafka | 9092 | Event bus |
| Redis | 6379 | Cache |
| Milvus | 19530 | Vector DB |

---

## Kafka Topics

| Topic | Producer | Consumers | Purpose |
|-------|----------|-----------|---------|
| business.events | Business Service | Network Service | Manual business creation events |
| transaction.events | Transaction Service | Network Service, Payment Service | Transaction + auto-creation triggers |
| search.index.requests | Network Service | Search Service | Indexing triggers for new nodes |

---

## Decision Points

### When Does Entity Resolution Run?
1. Manual business creation (always)
2. Transaction with unknown business (always)
3. Manual merge request from admin (on-demand)

### When Are New Nodes Created?
1. Entity Resolution confidence < 0.85 (no match found)
2. User overrides duplicate warning (manual)

### When Is Elasticsearch Updated?
1. New node created in Neo4j
2. Network Service publishes to `search.index.requests`
3. Search Service indexes asynchronously

---

## Key Metrics

### Entity Resolution Accuracy
- Target: 99.99% accuracy
- False Positives (incorrect match): <0.01%
- False Negatives (missed duplicate): <0.01%

### Latency Targets
- Manual business creation: <200ms
- Transaction auto-creation: <300ms
- Search query (cached): <5ms
- Search query (uncached): <100ms

### Throughput
- Business creation: 4.6 events/sec (average), 46 events/sec (peak)
- Entity Resolution: 13 requests/sec capacity
- Search queries: 100 req/sec target

---

## Common Scenarios

### Scenario 1: User Creates Duplicate Business
1. User creates "Acme Corporation"
2. Entity Resolution finds existing "Acme Corp" (confidence: 0.95)
3. Network Service updates existing node
4. User sees: "Business already exists, linked to your account"

### Scenario 2: User Creates Unique Business
1. User creates "Unique Startup LLC"
2. Entity Resolution finds no match (confidence: 0.12)
3. Network Service creates new node
4. Search Service indexes
5. Business searchable immediately

### Scenario 3: Transaction Creates Auto Node
1. User pays "New Vendor Inc"
2. Payment approved
3. Entity Resolution confirms no duplicate
4. Network Service creates node + relationship
5. Business appears in network map

### Scenario 4: Transaction with Existing Business
1. User pays "Acme Corp"
2. Entity Resolution finds match (confidence: 0.98)
3. Network Service updates relationship weight
4. No new node created

---

## Troubleshooting

### Issue: Duplicate Businesses Created
**Root Cause**: Entity Resolution confidence < 0.85
**Solution**: 
1. Check fuzzy search results from Search Service
2. Review ML model predictions
3. Adjust confidence threshold
4. Retrain model with feedback

### Issue: Businesses Not Searchable
**Root Cause**: Search Service not consuming from Kafka
**Solution**:
1. Check Kafka consumer lag
2. Verify `search.index.requests` topic
3. Check Elasticsearch indexing errors

### Issue: Slow Entity Resolution
**Root Cause**: Search Service fuzzy queries timeout
**Solution**:
1. Check Elasticsearch cluster health
2. Optimize fuzzy query parameters
3. Scale Search Service horizontally

---

## Summary

**Two Workflows, One Architecture:**

1. **Manual Creation**: User-driven, explicit business addition
2. **Transaction Auto-Creation**: Transaction-driven, implicit business discovery

**Core Pattern**: 
- Network Service orchestrates
- Entity Resolution determines duplicate/new
- Search Service provides fuzzy matching
- Neo4j stores graph
- Elasticsearch enables search

**Key Innovation**: Entity Resolution calls Search Service (not Elasticsearch directly) for better encapsulation and reusability.