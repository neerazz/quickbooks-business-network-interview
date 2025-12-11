# Implementation Plan

## Prerequisites and Notes for Implementers

**Before Starting Any Task:**

1. Read the full task description including all sub-tasks
2. Check the "Acceptance Criteria" section for each task
3. Verify all dependencies are complete before starting
4. Create a feature branch: `feature/{task-number}-{short-description}`
5. All tests must pass before marking a task complete
6. Update documentation in the same PR as code changes

**Technology References:**

- Java 17, Spring Boot 3.2.x
- PostgreSQL 15 (NOT Cassandra)
- Neo4j 5.x for graph database
- Elasticsearch 8.11.x for search
- Apache Kafka 3.6.x for messaging
- Redis 7.x for caching
- Vector DB (Pinecone/Milvus) for semantic search
- Python 3.11 + FastAPI for AI Service
- React 18 with TypeScript for frontend

**Property Test References:**

- All property numbers reference the Correctness Properties section in design.md
- Use jqwik library for property-based testing
- Minimum 100 iterations per property test

---

## Phase 1: Project Setup and Infrastructure

- [x] 1. Initialize Project Structure and Version Control

  **Prerequisites**: Git installed, GitHub access
  
  **Acceptance Criteria**:
  - All directories exist as specified
  - pom.xml files compile without errors
  - .gitignore excludes build artifacts
  
  - [x] 1.1 Create root project directory and initialize Git

    - Commands: mkdir quickbooks-business-network && cd quickbooks-business-network && git init
  
  - [x] 1.2 Create directory structure

    - Create directories:
      - .github/workflows/
      - .kiro/specs/quickbooks-business-network/
      - config/
      - docs/architecture/, docs/api/, docs/setup/, docs/user-guide/, docs/operations/
      - scripts/db/
      - src/services/gateway/, src/services/auth/, src/services/business/
      - src/services/network/, src/services/search/, src/services/transaction/
      - src/services/entity-resolution/, src/services/ingest-flow/
      - src/services/resolve-flow/, src/services/training-consumer/
      - src/services/graph-rag/, src/services/data-generator/, src/services/sla-metrics/
      - src/ai-service/, src/frontend/react-ui/
    - _Requirements: 29.1, 29.2, 29.3_
  
  - [x] 1.3 Create parent pom.xml at project root

    - groupId: com.quickbooks, artifactId: quickbooks-business-network
    - Java 17, Spring Boot 3.2.x parent
    - Dependencies: spring-boot-starter-web, spring-boot-starter-data-jpa,
      spring-boot-starter-data-neo4j, spring-boot-starter-data-elasticsearch,
      spring-kafka, spring-boot-starter-data-redis, spring-boot-starter-actuator,
      lombok, jqwik (test), junit-jupiter (test), mockito-core (test), testcontainers (test)
    - _Requirements: 29.4_
  
  - [x] 1.4 Create service pom.xml files for each service in src/services/

    - _Requirements: 29.1_
  
  - [x] 1.5 Create .gitignore and .env.example files

    - _Requirements: 27.4_
  
  - [x] 1.6 Create root README.md with project overview, architecture diagram, prerequisites, quick start

    - _Requirements: 29.5_

- [x] 2. Create Docker Compose Infrastructure

  **Prerequisites**: Docker and Docker Compose installed
  
  **Acceptance Criteria**:
  - docker-compose up starts all containers
  - All health checks pass
  - Services can communicate
  
  - [x] 2.1 Create docker-compose.yml at project root

    - Services: postgres:15, neo4j:5, elasticsearch:8.11.0, zookeeper, kafka, redis:7, milvus (vector DB)
    - Configure ports: PostgreSQL 5432, Neo4j 7474/7687, ES 9200, Kafka 9092, Redis 6379, Milvus 19530
    - Add health checks for all services
    - _Requirements: 29.4, 35.2_
  
  - [x] 2.2 Create scripts/db/init-postgres.sql

    - Tables: quickbooks_events, outbox, training_examples, user_feedback, audit_log, sla_metrics
    - Create indexes for performance
    - _Requirements: 16.1, 21.5, 47.4_
  
  - [x] 2.3 Create scripts/db/init-neo4j.cypher

    - Constraints: business_id uniqueness
    - Indexes: canonical_name, address_state, category
    - Full-text index for search fallback
    - _Requirements: 18.5_
  
  - [x] 2.4 Create scripts/db/init-kafka.sh

    - Create topics: quickbooks.events.invoice, quickbooks.events.payment,
      quickbooks.events.vendor, quickbooks.events.customer, entity.updated, entity.merged
    - _Requirements: 16.3_
  
  - [x] 2.5 Create scripts/db/init-elasticsearch.sh

    - Create business_entities index with ngram analyzer
    - Field mappings for id, canonical_name, category, state, city, aliases
    - _Requirements: 19.2_
  
  - [x] 2.6 Create scripts/setup.sh for environment setup

    - _Requirements: 27.1, 27.2_
  
  - [x] 2.7 Test infrastructure setup - verify all containers start and communicate

- [x] 3. Create Shared Domain Models Module

  **Prerequisites**: Task 1 complete
  
  **Acceptance Criteria**:
  - All domain classes compile
  - Unit tests pass
  - Classes are serializable to JSON
  
  - [x] 3.1 Create shared-models module in src/services/shared-models/

  - [x] 3.2 Create domain model classes:

    - BusinessEntity, Address, Relationship, RelationshipType enum (VENDOR, CLIENT)
    - QuickBooksEvent, EventType enum (INVOICE, PAYMENT, VENDOR_ADD, CUSTOMER_ADD)
    - ProcessingStatus enum (PENDING, PROCESSING, PUBLISHED, FAILED)
    - _Requirements: 1.1, 3.4_
  
  - [x] 3.3 Create Entity Resolution models:

    - RawBusinessData, StandardizedEntity, BlockingKey
    - FeatureVector (7 fields), ClassificationResult, EntityResolutionResult
    - _Requirements: 22.1_
  
  - [x] 3.4 Create Kafka event classes:

    - BusinessCreatedEvent, BusinessUpdatedEvent, BusinessDeletedEvent
    - TransactionProcessedEvent, EntityResolvedEvent, EntityMergedEvent
    - _Requirements: 16.2_
  
  - [x] 3.5 Create validation: ValidBusinessName annotation and BusinessNameValidator

    - Allow: alphanumeric, spaces, hyphens, apostrophes, periods, ampersands
    - _Requirements: 3.1_
  
  - [x] 3.6 Write unit tests for domain models

    - _Requirements: 3.1_

---

## Phase 2: Core Services Implementation

- [x] 4. Implement Business Service

  **Prerequisites**: Tasks 1, 2, 3 complete
  
  **Acceptance Criteria**:
  - All REST endpoints return correct responses
  - Data persists to PostgreSQL and Neo4j
  - Unit tests pass with 80%+ coverage
  
  - [x] 4.1 Create Business Service Spring Boot application (port 8081)

    - Configure PostgreSQL and Neo4j connections
    - _Requirements: 3.2, 3.3_
  
  - [x] 4.2 Create repository layer:

    - BusinessJpaRepository (PostgreSQL)
    - BusinessNeo4jRepository (Neo4j with custom Cypher queries)
    - _Requirements: 3.2_
  
  - [x] 4.3 Create BusinessService with CRUD operations:

    - createBusiness, getBusiness, updateBusiness, deleteBusiness
    - listBusinesses (paginated), simulateSession (POC)
    - _Requirements: 10.3, 10.4_
  
  - [x] 4.4 Create BusinessController REST endpoints

    - _Requirements: 3.2, 10.2_
  
  - [x] 4.5 Implement business name validation (non-empty, valid characters only)

    - _Requirements: 3.1_
  
  - [x] 4.6 Write Property Test: Entity Name Validation (Property 5)

    - Test empty/whitespace rejection, valid name acceptance
    - **Validates: Requirements 3.1**
  
  - [x] 4.7 Implement Kafka event publishing for business lifecycle

    - _Requirements: 16.2_
  
  - [x] 4.8 Write Property Test: Entity Creation Idempotence (Property 6)

    - Test first add creates node, subsequent add creates relationship to existing
    - **Validates: Requirements 3.2, 3.3**
  
  - [x] 4.9 Write unit tests (1 positive, 2 negative per method)

    - _Requirements: 3.2, 3.3_

- [x] 5. Implement Network Service
  
  **Prerequisites**: Task 4 complete
  
  **Acceptance Criteria**:
  - Network map returns all direct relationships
  - Indirect relationships compute correctly up to 3 hops

  - Relationship CRUD operations work correctly
  
  - [x] 5.1 Create Network Service Spring Boot application (port 8082)
    - Configure Neo4j, Elasticsearch, Redis

    - _Requirements: 1.1, 3.3, 3.4_
  
  - [x] 5.2 Create Neo4j repository for relationships with Cypher queries
    - findDirectRelationships, findIndirectRelationships, findShortestPath

    - _Requirements: 1.1, 2.4_

  - [x] 5.3 Create NetworkService:

    - getNetworkMap, getDirectRelationships, getIndirectRelationships, findShortestPath

    - _Requirements: 1.1, 2.3, 2.4_
  
  - [x] 5.4 Create RelationshipService:
    - createRelationship (init weight to 0.00), deleteRelationship (preserve nodes)
    - _Requirements: 3.3, 3.4, 3.5, 7.1, 7.2_
  
  - [x] 5.5 Write Property Test: Network Map Retrieval Completeness (Property 1)
    - **Validates: Requirements 1.1, 2.3**
  
  - [x] 5.6 Write Property Test: New Relationship Weight Initialization (Property 7)
    - **Validates: Requirements 3.5**
  
  - [x] 5.7 Write Property Test: Indirect Relationship Boundary (Property 4)
    - **Validates: Requirements 2.4, 20.3**

  - [x] 5.8 Write Property Test: Relationship Removal Preservation (Property 15)

    - **Validates: Requirements 7.1, 7.2**

  - [x] 5.9 Write Property Test: Referential Integrity Preservation (Property 13)

    - Test no orphaned nodes or edges after operations

    - **Validates: Requirements 4.5, 7.5**
  
  - [x] 5.10 Implement Redis caching (network maps 10min, relationships 5min)
    - _Requirements: 5.2, 5.3_
  
  - [x] 5.11 Create NetworkController REST endpoints

    - _Requirements: 1.1, 7.4_
  
  - [x] 5.12 Write unit tests
    - _Requirements: 1.1, 2.4, 7.1_

- [x] 6. Checkpoint: Core Services Tests

  - [ ] All unit tests pass
  - [x] All property tests pass (100 iterations each)

  - [x] Code coverage >= 80%

  Ask user if any questions arise before proceeding.

---

## Phase 3: Search and Transaction Services

- [ ] 7. Implement Search Service

  **Prerequisites**: Tasks 4, 5 complete
  
  - [x] 7.1 Create Search Service Spring Boot application (port 8083)

    - _Requirements: 2.1_

  - [x] 7.2 Create Elasticsearch repository with BusinessDocument

    - _Requirements: 2.1_
  
  - [x] 7.3 Implement SearchService with ngram matching

    - _Requirements: 2.1, 2.2_

  - [x] 7.4 Implement GraphTraversalService for connected nodes

    - _Requirements: 20.3, 20.4_

  - [x] 7.5 Write Property Test: Search Results Completeness (Property 2)

    - **Validates: Requirements 2.1**

  - [x] 7.6 Write Property Test: Relationship Type Filtering (Property 3)
    - **Validates: Requirements 2.2**

  - [x] 7.7 Implement Kafka consumer for index updates

    - _Requirements: 19.3_

  - [x] 7.8 Create SearchController REST endpoints
    - _Requirements: 2.1, 20.1_
  
  - [x] 7.9 Write unit tests

- [x] 8. Implement Transaction Service

  **Prerequisites**: Tasks 4, 5 complete
  
  - [x] 8.1 Create Transaction Service Spring Boot application (port 8084)
    - _Requirements: 6.1_
  
  - [x] 8.2 Create Transaction repository
    - _Requirements: 6.1_
  
  - [x] 8.3 Implement TransactionService:
    - recordTransaction, recalculateWeight
    - _Requirements: 6.1, 6.2, 6.4_
  
  - [x] 8.4 Write Property Test: Transaction Volume Aggregation (Property 14)
    - **Validates: Requirements 6.1, 6.4**
  
  - [x] 8.5 Create TransactionController REST endpoints
    - _Requirements: 6.1_
  
  - [x] 8.6 Write unit tests

---

## Phase 4: Entity Resolution Service

- [x] 9. Implement Entity Resolution - Standardize Stage

  **Prerequisites**: Task 3 complete
  
  - [x] 9.1 Create Entity Resolution Service (port 8085)
    - _Requirements: 22.1_
  
  - [x] 9.2 Create AbbreviationConfig with NAME_ABBREVIATIONS and ADDRESS_ABBREVIATIONS
    - _Requirements: 23.2, 23.3_
  
  - [x] 9.3 Implement StandardizeStage:
    - lowercase, collapse whitespace, expand abbreviations, remove punctuation
    - _Requirements: 23.1, 23.2, 23.4, 23.5_
  
  - [x] 9.4 Implement address standardization
    - _Requirements: 23.3_
  
  - [x] 9.5 Write Property Test: Standardization Consistency (Property 18)
    - **Validates: Requirements 23.1, 23.2**
  
  - [x] 9.6 Write Property Test: Raw Data Preservation (Property 19)
    - **Validates: Requirements 23.5**
  
  - [x] 9.7 Write Property Test: Address Normalization (Property 20)
    - Test street type, directional, and unit designator expansion
    - **Validates: Requirements 23.3**
  
  - [x] 9.8 Write unit tests

- [x] 10. Implement Entity Resolution - Block Stage
  
  - [x] 10.1 Implement BlockStage:
    - generateBlockingKeys (name prefix, state, composite)
    - assignToBlocks, subdivideBlock (when >1000)
    - _Requirements: 24.1, 24.2, 24.4_
  
  - [x] 10.2 Write Property Test: Blocking Key Generation (Property 21)
    - **Validates: Requirements 24.1, 24.2**
  
  - [x] 10.3 Write Property Test: True Match Block Co-location (Property 22)
    - **Validates: Requirements 24.3**
  
  - [x] 10.4 Write Property Test: Large Block Subdivision (Property 23)
    - **Validates: Requirements 24.4**
  
  - [x] 10.5 Write unit tests

- [x] 11. Implement Entity Resolution - Compare Stage

  - [x] 11.1 Implement similarity algorithms:
    - LevenshteinDistance, JaroWinklerSimilarity, TokenSortRatio
    - _Requirements: 25.1_
  
  - [x] 11.2 Implement CompareStage:
    - Generate 7-feature FeatureVector (levenshtein, jaro-winkler, token, address, category, legal type, state)
    - _Requirements: 25.1, 25.2, 25.3_
  
  - [x] 11.3 Write Property Test: Similarity Score Range (Property 24)
    - **Validates: Requirements 25.4**
  
  - [x] 11.4 Write Property Test: Feature Vector Completeness (Property 25)
    - **Validates: Requirements 25.3, 25.5**
  
  - [x] 11.5 Write unit tests

- [x] 12. Implement Entity Resolution - Classify Stage

  - [x] 12.1 Create AIServiceClient (HTTP to Python service)
    - _Requirements: 26.1_
  
  - [x] 12.2 Implement ClassifyStage:
    - Apply thresholds: >=0.85 match, 0.50-0.85 review, <0.50 no match
    - _Requirements: 26.2, 26.3, 26.4_
  
  - [x] 12.3 Write Property Test: Classification Output Validity (Property 26)
    - **Validates: Requirements 22.4, 26.2**
  
  - [x] 12.4 Write Property Test: Low Confidence Flagging (Property 12)
    - **Validates: Requirements 4.4, 26.4**
  
  - [x] 12.5 Write unit tests

- [x] 13. Implement Complete Entity Resolution Pipeline

  - [x] 13.1 Create EntityResolutionPipeline:
    - Orchestrate all 4 stages
    - _Requirements: 22.1, 22.2, 22.3, 22.4, 22.5_
  
  - [x] 13.2 Write Property Test: Entity Resolution Case Insensitivity (Property 8)
    - **Validates: Requirements 4.1, 15.2**
  
  - [x] 13.3 Write Property Test: Abbreviation Handling (Property 9)
    - **Validates: Requirements 4.1, 15.3**
  
  - [x] 13.4 Write Property Test: Legal Entity Type Differentiation (Property 10)
    - **Validates: Requirements 15.4**
  
  - [x] 13.5 Create REST controllers (EntityResolutionController, FeedbackController)
    - _Requirements: 13.1, 13.2_
  
  - [x] 13.6 Write unit tests

- [x] 14. Implement AI Service (Python FastAPI)

  **Prerequisites**: Task 12 complete
  
  **Acceptance Criteria**:
  - AI Service runs on port 8000
  - ML model classifies entity pairs correctly
  - API responds within 100ms at p95
  
  - [x] 14.1 Create Python FastAPI application structure

    - app/main.py, app/model/, app/api/
    - _Requirements: 26.1_
  
  - [x] 14.2 Implement ML model loading and inference

    - Random Forest or XGBoost model for classification
    - _Requirements: 26.1, 26.5_
  
  - [x] 14.3 Create /api/v1/classify endpoint

    - Accept feature vectors, return match_probability
    - _Requirements: 26.2_
  
  - [x] 14.4 Create Dockerfile for AI Service

    - Python 3.11, FastAPI, scikit-learn/XGBoost
    - _Requirements: 29.4_
  
  - [x] 14.5 Write unit tests (pytest)

    - _Requirements: 26.2_

- [x] 15. Checkpoint: Entity Resolution Tests

  - [ ] All unit tests pass
  - [ ] All property tests pass (100 iterations each)
  - [ ] Code coverage >= 80% for entity resolution module
  
  Ask user if any questions arise before proceeding.

---

## Phase 5: Data Processing Services

- [x] 16. Implement Ingest Flow Service

  - [x] 16.1 Create Ingest Flow Service (port 8087)

    - _Requirements: 16.1_
  
  - [x] 16.2 Implement IngestService with transactional outbox pattern

    - _Requirements: 16.1, 16.5_
  
  - [x] 16.3 Implement OutboxPollingScheduler (poll 100ms, retry with backoff)

    - _Requirements: 16.2, 16.4_
  
  - [x] 16.4 Write Property Test: Ingest Flow Atomicity (Property 31)

    - **Validates: Requirements 16.5**
  
  - [x] 16.5 Write Property Test: Kafka Topic Routing (Property 32)

    - **Validates: Requirements 16.3**
  
  - [x] 16.6 Create IngestController REST endpoints

    - _Requirements: 16.1_
  
  - [x] 16.7 Write unit tests

- [x] 17. Implement Resolve Flow Service

  - [x] 17.1 Create Resolve Flow Service (port 8088)

    - _Requirements: 17.1_
  
  - [x] 17.2 Implement Kafka consumer for all quickbooks.events.* topics

    - _Requirements: 17.1, 17.2_
  
  - [x] 17.3 Implement ResolveFlowService:

    - Process events, call Entity Resolution, persist to Neo4j/ES
    - _Requirements: 17.3, 17.4, 17.5_
  
  - [x] 17.4 Implement MergeService:

    - Merge aliases, sum transaction volumes, transfer relationships
    - _Requirements: 4.3_
  
  - [x] 17.5 Write Property Test: Entity Merge Consolidation (Property 11)

    - **Validates: Requirements 4.3, 17.5**
  
  - [x] 17.6 Implement Elasticsearch sync

    - _Requirements: 19.1, 19.3_
  
  - [x] 17.7 Implement Vector DB sync for embeddings

    - _Requirements: 35.1, 35.4_
  
  - [x] 17.8 Write unit tests

- [x] 18. Implement Data Generator Service

  - [x] 18.1 Create Data Generator Service (port 8090)

    - _Requirements: 9.1_
  
  - [x] 18.2 Create data files (business_names.txt, street_names.txt, cities.txt, categories.txt)

  - [x] 18.3 Implement BusinessGenerator

    - _Requirements: 9.1, 9.3, 9.4_
  
  - [x] 18.4 Implement MessinessInjector:

    - 30% typos, 40% abbreviations, 50% case variations
    - _Requirements: 9.3, 9.4_
  
  - [x] 18.5 Implement EventGenerator with power-law distribution

    - _Requirements: 9.2_
  
  - [x] 18.6 Write Property Test: Data Generator Messiness (Property 36)

    - **Validates: Requirements 9.3, 9.4**
  
  - [x] 18.7 Create DataGeneratorController REST endpoints

    - _Requirements: 9.5_
  
  - [x] 18.8 Write unit tests

- [x] 19. Implement Automated Database Population

  - [x] 19.1 Create StartupDataPopulator (ApplicationRunner)

    - Detect empty databases, trigger population
    - _Requirements: 30.1, 30.2, 30.3, 30.4_
  
  - [x] 19.2 Write Property Test: Automated Population Completeness (Property 35)

    - **Validates: Requirements 30.1, 30.2, 9.1, 9.2**
  
  - [x] 19.3 Write unit tests

- [x] 20. Checkpoint: Data Processing Tests

  - [ ] All unit tests pass
  - [ ] All property tests pass
  - [ ] End-to-end data flow works
  
  Ask user if any questions arise before proceeding.

---

## Phase 6: Advanced Services

- [x] 21. Implement Graph RAG Service

  - [x] 21.1 Create Graph RAG Service (port 8086)

    - _Requirements: 34.1_
  
  - [x] 21.2 Implement QueryParserService:

    - Extract intent, entities, filters from natural language
    - _Requirements: 34.2, 34.3_
  
  - [x] 21.3 Write Property Test: Natural Language Query Parsing (Property 27)

    - **Validates: Requirements 34.1, 34.3**
  
  - [x] 21.4 Implement VectorSearchService

    - _Requirements: 35.3, 36.1, 36.2_
  
  - [x] 21.5 Implement GraphTraversalService with filters

    - _Requirements: 36.3, 36.4_
  
  - [x] 21.6 Write Property Test: Graph Traversal Depth Limit (Property 38)

    - **Validates: Requirements 37.2, 20.3**
  
  - [x] 21.7 Write Property Test: Context Resolution (Property 28)

    - **Validates: Requirements 37.1**
  
  - [x] 21.8 Implement HybridRankingService (60% semantic + 40% graph)

    - _Requirements: 36.5_
  
  - [x] 21.9 Write Property Test: Graph RAG Result Combination (Property 29)

    - Test results combine Vector DB semantic similarity with Neo4j graph connectivity
    - **Validates: Requirements 34.4, 36.4**
  
  - [x] 21.10 Write Property Test: Graph RAG Response Completeness (Property 30)

    - **Validates: Requirements 34.5, 38.3**
  
  - [x] 21.11 Create GraphRAGController (REST only, NO WebSocket)

    - _Requirements: 38.1, 38.2, 38.5_
  
  - [x] 21.12 Write unit tests

- [x] 22. Implement Training Consumer Service

  - [x] 22.1 Create Training Consumer Service (port 8089)

    - _Requirements: 21.1_
  
  - [x] 22.2 Implement TrainingDataConsumer

    - _Requirements: 21.2, 21.3_
  
  - [x] 22.3 Implement FeedbackIncorporationService (3x weight for user feedback)

    - _Requirements: 21.4, 14.2_
  
  - [x] 22.4 Write Property Test: User Feedback Incorporation (Property 33)

    - **Validates: Requirements 13.3, 13.4, 14.1**
  
  - [x] 22.5 Write unit tests

- [x] 23. Implement SLA Metrics Service

  - [x] 23.1 Create SLA Metrics Service (port 8091)
    - _Requirements: 11.1_
  
  - [x] 23.2 Implement MetricsCollectionService
    - _Requirements: 11.1, 11.3_
  
  - [x] 23.3 Implement PercentileCalculationService (p50, p75, p90, p95, p99)
    - _Requirements: 11.2_
  
  - [x] 23.4 Write Property Test: SLA Metrics Accuracy (Property 34)
    - **Validates: Requirements 11.1, 11.2**
  
  - [x] 23.5 Implement AlertService (threshold breach for 5 consecutive minutes)
    - _Requirements: 11.5_
  
  - [x] 23.6 Create SLAMetricsController REST endpoints
    - _Requirements: 11.4_
  
  - [x] 23.7 Write unit tests

- [x] 24. Implement Export Service
  
  - [x] 24.1 Add ExportService to Network Service
    - _Requirements: 8.1, 8.2_
  
  - [x] 24.2 Implement JSON and CSV serializers
    - _Requirements: 8.3_
  
  - [x] 24.3 Write Property Test: Export Completeness (Property 16)
    - **Validates: Requirements 8.1, 8.5**
  
  - [x] 24.4 Write Property Test: Export Format Validity (Property 17)
    - **Validates: Requirements 8.3**
  
  - [x] 24.5 Write unit tests

- [x] 25. Implement Audit Logging Service

  - [x] 25.1 Create AuditLogService in shared-models or gateway
    - _Requirements: 47.1, 47.2, 47.3_
  
  - [x] 25.2 Implement audit logging for entity CRUD operations
    - Log: action, entity_id, user_id, timestamp, before_state, after_state
    - _Requirements: 47.1, 47.4_
  
  - [x] 25.3 Implement audit logging for relationship operations
    - _Requirements: 47.2_
  
  - [x] 25.4 Implement audit logging for entity resolution merges
    - _Requirements: 47.3_
  
  - [x] 25.5 Create GET /api/v1/audit-log endpoint with filtering
    - _Requirements: 47.5_
  
  - [x] 25.6 Write unit tests

- [ ] 26. Checkpoint: Advanced Services Tests

  - [ ] All unit tests pass
  - [ ] All property tests pass
  - [ ] Integration tests pass
  
  Ask user if any questions arise before proceeding.

---

## Phase 7: Frontend and API Gateway

- [x] 27. Implement React UI Frontend

  - [x] 27.1 Create React application with TypeScript

    - Install: react-query, d3, axios, tailwindcss
    - _Requirements: 1.2, 2.1_
  
  - [x] 27.2 Create NetworkGraph component (D3.js force-directed graph)

    - _Requirements: 1.2, 1.3_
  
  - [x] 27.3 Create BusinessSearch component with autocomplete

    - _Requirements: 2.1_
  
  - [x] 27.4 Create RelationshipManager component

    - _Requirements: 3.3, 3.4, 7.1, 7.4_
  
  - [x] 27.5 Create UserSelector component (POC)

    - _Requirements: 10.1, 10.2_
  
  - [x] 27.6 Create GraphRAGChat component

    - _Requirements: 34.5, 38.1_
  
  - [x] 27.7 Create Feedback components

    - _Requirements: 13.1, 13.2_
  
  - [x] 27.8 Write component tests (Jest + React Testing Library)

---

## Phase 8: Project Cleanup & Consolidation (Maintenance)

- [ ] 28. Consolidate Startup Scripts
  
  **Prerequisites**: None
  
  **Acceptance Criteria**:
  - `start.ps1` and `start.sh` exist and work identically
  - `start-all.ps1` and `start-complete.ps1` are removed
  - End-to-end validation is integrated
  
  - [ ] 28.1 Implement `scripts/start.ps1` (Windows)
    - Incorporate logic from `start-complete.ps1` and `validate-setup.ps1`
    - _Requirements: 57.1, 57.2, 57.3_
  
  - [ ] 28.2 Implement `scripts/start.sh` (Linux/Mac)
    - Feature parity with PowerShell script
    - _Requirements: 57.1_
  
  - [ ] 28.3 Remove legacy scripts
  
- [ ] 29. Project Root Cleanup
  
  **Acceptance Criteria**:
  - Root directory is clean of logs
  - .gitignore is updated
  
  - [ ] 29.1 Update `.gitignore`
    - Ignore `/logs`, `*.log` (root), `maven-wrapper.zip`
  
  - [ ] 29.2 Archive/Move existing logs to `logs/` directory

- [x] 28. Implement API Gateway
  
  - [x] 28.1 Create API Gateway Service (port 8080)
    - _Requirements: 5.4_
  
  - [x] 28.2 Configure routes to all services
  
  - [x] 28.3 Implement Redis-based rate limiting (100/min standard, 10/min expensive)
    - _Requirements: 48.1, 48.2, 48.3_
  
  - [x] 28.4 Configure CORS
  
  - [x] 28.5 Add correlation ID filter
    - _Requirements: 43.2_
  
  - [x] 28.6 Implement timeout configuration for external calls
    - Entity Resolution: 30s, Neo4j: 10s, Elasticsearch: 5s, Vector DB: 5s
    - _Requirements: 50.1, 50.2, 50.3, 50.4, 50.5_
  
  - [x] 28.7 Write unit tests

---

## Phase 8: Documentation and CI/CD

- [x] 29. Create Documentation
  
  - [x] 29.1 Create docs/setup/getting-started.md
    - _Requirements: 27.1, 27.2, 27.3_
  
  - [x] 29.2 Create docs/setup/database-config.md
    - _Requirements: 27.3_
  
  - [x] 29.3 Create docs/setup/troubleshooting.md
    - _Requirements: 27.5_
  
  - [x] 29.4 Create docs/user-guide/usage.md
    - _Requirements: 28.1, 28.2_
  
  - [x] 29.5 Create docs/api/openapi-v1.yaml
    - _Requirements: 27.2, 49.4_
  
  - [x] 29.6 Create docs/architecture/overview.md with diagrams
    - _Requirements: 28.4, 40.1_
  
  - [x] 29.7 Create docs/architecture/data-flow.md
    - _Requirements: 40.2_
  
  - [x] 29.8 Create docs/operations/runbooks.md
    - _Requirements: 43.5_

- [x] 30. Set up CI/CD Pipeline
  
  - [x] 30.1 Create .github/workflows/ci.yml
    - Build, test, property tests, coverage, lint
    - _Requirements: 29.3_
  
  - [x] 30.2 Create .github/workflows/cd.yml
    - Docker build and push
    - _Requirements: 29.3_
  
  - [x] 30.3 Create .github/pull_request_template.md
  
  - [x] 30.4 Write integration tests for CI/CD

- [x] 31. Multi-AZ Deployment Configuration (Documentation Only)
  
  - [x] 31.1 Document PostgreSQL Multi-AZ setup
    - _Requirements: 31.1, 31.2, 31.3, 31.4_
  
  - [x] 31.2 Document Neo4j Fabric sharding
    - _Requirements: 32.1, 32.2, 32.3_
  
  - [x] 31.3 Document Elasticsearch cluster setup
    - _Requirements: 33.1, 33.2_
  
  - [x] 31.4 Document Kafka Multi-AZ deployment
    - _Requirements: 33.3, 33.4, 33.5_
  
  - [x] 31.5 Write Property Test: Multi-AZ Replication (Property 37) - Design validation
    - **Validates: Requirements 31.3, 31.4**

---

## Phase 9: Final Integration and Testing

- [x] 32. End-to-End Integration Testing
  
  - [x] 32.1 Create integration test suite for full user flows
  
  - [x] 32.2 Create performance tests (verify SLA targets)
  
  - [x] 32.3 Run full test suite
  
  - [x] 32.4 Fix any failing tests
  
  - [x] 32.5 Code review and cleanup
    - _Requirements: 44.1, 44.2, 44.3, 44.4, 44.5_

- [x] 33. Final Checkpoint: Production Readiness
  - [x] All 38 property tests pass with 100 iterations each
  - [x] All unit tests pass (minimum 80% coverage)
  - [x] All integration tests pass
  - [x] Performance targets met
  - [x] Documentation complete and accurate
  - [x] No critical or high severity bugs
  - [x] Code review completed
  - [x] CI/CD pipeline green
  
  System is ready for deployment!

---

## Phase 10: Data Availability and Seeding

- [x] 34. Create Data Availability Script

  **Prerequisites**: All infrastructure (Docker) and core microservices implemented
  
  **Acceptance Criteria**:
  - Script checks all infrastructure services are healthy
  - Script checks required microservices are running
  - Script seeds test data through the Ingest API
  - Script verifies data flows through the complete pipeline
  - Script works on both Windows (PowerShell) and Linux/macOS (Bash)
  
  - [x] 34.1 Create PowerShell script (`scripts/seed-data.ps1`)
    - Infrastructure health checks (PostgreSQL, Neo4j, Elasticsearch, Kafka, Redis)
    - Microservice health checks (Gateway, Ingest Flow, Resolve Flow, Entity Resolution)
    - Test data generation with messy business data for Entity Resolution testing
    - API-based data seeding via POST /api/v1/ingest/events/batch
    - Pipeline verification (PostgreSQL → Kafka → Neo4j → Elasticsearch)
    - _Requirements: 30.1, 30.2, 30.3, 30.4_
  
  - [x] 34.2 Create Bash script (`scripts/seed-data.sh`)
    - Equivalent functionality for Linux/macOS environments
    - _Requirements: 30.1, 30.2, 30.3, 30.4_
  
  - [ ] 34.3 Test script execution flow
    - Verify infrastructure checks work correctly
    - Verify microservice checks work correctly
    - Verify data seeding succeeds
    - Verify pipeline flow verification works

---

## Appendix: Property Test to Requirement Mapping

| Property | Name | Requirements |
|----------|------|--------------|
| 1 | Network Map Retrieval Completeness | 1.1, 2.3 |
| 2 | Search Results Completeness | 2.1 |
| 3 | Relationship Type Filtering | 2.2 |
| 4 | Indirect Relationship Boundary | 2.4, 20.3 |
| 5 | Entity Name Validation | 3.1 |
| 6 | Entity Creation Idempotence | 3.2, 3.3 |
| 7 | New Relationship Weight Initialization | 3.5 |
| 8 | Entity Resolution Case Insensitivity | 4.1, 15.2 |
| 9 | Entity Resolution Abbreviation Handling | 4.1, 15.3 |
| 10 | Legal Entity Type Differentiation | 15.4 |
| 11 | Entity Merge Consolidation | 4.3, 17.5 |
| 12 | Low Confidence Flagging | 4.4, 26.4 |
| 13 | Referential Integrity Preservation | 4.5, 7.5 |
| 14 | Transaction Volume Aggregation | 6.1, 6.4 |
| 15 | Relationship Removal Preservation | 7.1, 7.2 |
| 16 | Export Completeness | 8.1, 8.5 |
| 17 | Export Format Validity | 8.3 |
| 18 | Standardization Consistency | 23.1, 23.2 |
| 19 | Raw Data Preservation | 23.5 |
| 20 | Address Normalization | 23.3 |
| 21 | Blocking Key Generation | 24.1, 24.2 |
| 22 | True Match Block Co-location | 24.3 |
| 23 | Large Block Subdivision | 24.4 |
| 24 | Similarity Score Range | 25.4 |
| 25 | Feature Vector Completeness | 25.3, 25.5 |
| 26 | Classification Output Validity | 22.4, 26.2 |
| 27 | Graph RAG Query Parsing | 34.1, 34.3 |
| 28 | Graph RAG Context Resolution | 37.1 |
| 29 | Graph RAG Result Combination | 34.4, 36.4 |
| 30 | Graph RAG Response Completeness | 34.5, 38.3 |
| 31 | Ingest Flow Atomicity | 16.5 |
| 32 | Kafka Topic Routing | 16.3 |
| 33 | User Feedback Incorporation | 13.3, 13.4, 14.1 |
| 34 | SLA Metrics Accuracy | 11.1, 11.2 |
| 35 | Automated Population Completeness | 30.1, 30.2, 9.1, 9.2 |
| 36 | Data Generator Messiness | 9.3, 9.4 |
| 37 | Multi-AZ Data Replication | 31.3, 31.4 |
| 38 | Graph Traversal Depth Limit | 37.2, 20.3 |

---

## Appendix: Service Port Mapping

| Service | Port |
|---------|------|
| API Gateway | 8080 |
| Business Service | 8081 |
| Network Service | 8082 |
| Search Service | 8083 |
| Transaction Service | 8084 |
| Entity Resolution Service | 8085 |
| Graph RAG Service | 8086 |
| Ingest Flow Service | 8087 |
| Resolve Flow Service | 8088 |
| Training Consumer Service | 8089 |
| Data Generator Service | 8090 |
| SLA Metrics Service | 8091 |
| Auth Service | 8092 |
| AI Service (Python) | 8000 |
| PostgreSQL | 5432 |
| Neo4j (Bolt) | 7687 |
| Neo4j (HTTP) | 7474 |
| Elasticsearch | 9200 |
| Kafka | 9092 |
| Zookeeper | 2181 |
| Redis | 6379 |
| Milvus (Vector DB) | 19530 |
| React UI | 3000 |

---

## Phase 11: End-to-End Data Flow Validation

- [x] 35. Implement Data Ingestion UI Page

  **Prerequisites**: Phase 7 (Frontend) complete, Ingest Flow Service running
  
  **Acceptance Criteria**:
  - UI provides form to create QuickBooks events
  - Form validates all required fields
  - Success/error feedback displayed with event IDs
  - Link to Flow Monitor for tracking
  
  - [x] 35.1 Create DataIngestionPage component
    - Form with fields: event_type (dropdown), source_business_name, source_business_address, target_business_name, target_business_address, amount
    - Form validation: all fields required, amount must be positive number
    - Submit button calls POST /api/v1/ingest/events
    - _Requirements: 55.1, 55.2, 55.3_
  
  - [x] 35.2 Create ingestService API client
    - Function: ingestEvent(request: IngestEventRequest): Promise<IngestEventResponse>
    - Error handling with user-friendly messages
    - _Requirements: 55.3_
  
  - [x] 35.3 Add success/error feedback UI
    - Success: Display event_id and status with green checkmark
    - Error: Display error message with red indicator
    - Link to Flow Monitor: "Track progress →"
    - _Requirements: 55.4_
  
  - [x] 35.4 Add navigation menu item
    - Add "Data Ingestion" link to main navigation
    - Icon: upload or database icon
    - _Requirements: 55.1_
  
  - [ ] 35.5 Write component tests
    - Test form validation
    - Test successful submission
    - Test error handling
    - _Requirements: 55.2, 55.3, 55.4_

- [x] 36. Implement Flow Monitor Dashboard

  **Prerequisites**: Task 35 complete, all backend services running
  
  **Acceptance Criteria**:
  - Dashboard displays real-time metrics from all pipeline stages
  - Auto-refresh every 5 seconds
  - Visual indicators show pipeline health
  - Detailed views available for each stage
  
  - [x] 36.1 Create FlowMonitorPage component
    - Layout with 4 cards: PostgreSQL, Kafka, Neo4j, Elasticsearch
    - Each card shows: count, status indicator (green/yellow/red), last updated time
    - Auto-refresh using useEffect with 5-second interval
    - _Requirements: 56.1, 56.2, 56.3_
  
  - [x] 36.2 Create metrics API endpoints
    - Ingest Flow: GET /api/v1/ingest/metrics
    - Resolve Flow: GET /api/v1/resolve/metrics  
    - Search Service: GET /api/v1/search/metrics
    - Return counts by status, processing times, error counts
    - _Requirements: 56.1_
  
  - [x] 36.3 Implement metrics collection in Ingest Flow Service
    - Track: total_events, by_status counts, by_type counts
    - Expose via /api/v1/ingest/metrics endpoint
    - _Requirements: 56.1_
  
  - [ ] 36.4 Implement metrics collection in Resolve Flow Service
    - Track: events_consumed, entities_resolved, nodes_created, relationships_created
    - Expose via /api/v1/resolve/metrics endpoint
    - _Requirements: 56.1_
  
  - [ ] 36.5 Implement metrics collection in Search Service
    - Track: elasticsearch_documents, neo4j_nodes, sync_lag_seconds
    - Expose via /api/v1/search/metrics endpoint
    - _Requirements: 56.1_
  
  - [x] 36.6 Create visual pipeline status component
    - Display: PostgreSQL → Kafka → Neo4j → Elasticsearch
    - Color indicators: green (healthy), yellow (lag), red (errors)
    - Show counts for each stage
    - _Requirements: 56.3_
  
  - [ ] 36.7 Implement detailed stage views
    - Click on stage card to expand details
    - Show: recent events, error messages, processing times, sample data
    - _Requirements: 56.4_
  
  - [x] 36.8 Add error indicators and troubleshooting
    - Red indicators when errors detected
    - Display error messages with event IDs
    - Link to Kibana logs with trace_id
    - _Requirements: 56.5_
  
  - [ ] 36.9 Write component tests
    - Test metrics fetching
    - Test auto-refresh
    - Test status indicators
    - Test detailed views
    - _Requirements: 56.1, 56.2, 56.3, 56.4_

- [ ] 37. Create End-to-End Flow Integration Test

  **Prerequisites**: Tasks 35, 36 complete
  
  **Acceptance Criteria**:
  - Automated test validates complete pipeline flow
  - Test creates event via API and verifies it reaches Elasticsearch
  - Test runs in CI/CD pipeline
  - Test provides clear failure messages
  
  - [ ] 37.1 Create E2E test script
    - Script: `scripts/test-e2e-flow.sh` (Bash) and `scripts/test-e2e-flow.ps1` (PowerShell)
    - Steps: 1) POST event to Ingest API, 2) Wait for processing, 3) Verify in Elasticsearch, 4) Verify in Neo4j
    - _Requirements: 54.1, 54.2, 54.3, 54.4, 54.5_
  
  - [ ] 37.2 Implement test with retries and timeouts
    - Retry search up to 10 times with 3-second intervals (30s total)
    - Timeout if event not found after 30 seconds
    - Clear error messages indicating which stage failed
    - _Requirements: 54.2, 54.3, 54.4_
  
  - [ ] 37.3 Add verification checks
    - PostgreSQL: Event exists with status=PUBLISHED
    - Kafka: Message exists in appropriate topic
    - Neo4j: Node exists with canonical_name
    - Elasticsearch: Document exists and is searchable
    - _Requirements: 54.1, 54.2, 54.3, 54.4, 54.5_
  
  - [ ] 37.4 Create Playwright E2E test for UI flow
    - Test: User fills form → submits → sees success → navigates to Flow Monitor → sees updated counts
    - File: `src/frontend/react-ui/e2e/data-ingestion-flow.spec.ts`
    - _Requirements: 55.1, 55.2, 55.3, 55.4, 56.1, 56.2_
  
  - [ ] 37.5 Add E2E test to CI/CD pipeline
    - Run E2E test in GitHub Actions workflow
    - Fail build if E2E test fails
    - Generate test report with screenshots on failure
    - _Requirements: 54.1, 54.2, 54.3, 54.4, 54.5_

- [ ] 38. Create End-to-End Flow Documentation

  **Prerequisites**: Tasks 35, 36, 37 complete
  
  **Acceptance Criteria**:
  - Documentation explains complete data flow
  - Step-by-step guide for testing the flow
  - Troubleshooting guide for common issues
  - Architecture diagrams included
  
  - [ ] 38.1 Create end-to-end flow guide
    - File: `docs/user-guide/end-to-end-data-flow.md`
    - Sections: Overview, Architecture, Step-by-Step Guide, Verification, Troubleshooting
    - Include sequence diagram from design.md
    - _Requirements: 54.1, 54.2, 54.3, 54.4, 54.5_
  
  - [ ] 38.2 Create UI usage guide
    - File: `docs/user-guide/data-ingestion-ui.md`
    - Screenshots of Data Ingestion page and Flow Monitor
    - Example event data for testing
    - Expected results and timings
    - _Requirements: 55.1, 55.2, 55.3, 55.4_
  
  - [ ] 38.3 Create troubleshooting guide
    - File: `docs/setup/troubleshooting-data-flow.md`
    - Common issues: Event stuck in PENDING, Kafka not consuming, Neo4j not creating nodes, Elasticsearch not indexing
    - For each issue: Symptoms, Diagnosis steps, Resolution
    - _Requirements: 56.5_
  
  - [ ] 38.4 Update README with E2E flow section
    - Add "End-to-End Data Flow" section
    - Link to detailed documentation
    - Quick verification steps
    - _Requirements: 54.1, 54.2, 54.3, 54.4, 54.5_

- [ ] 39. Final End-to-End Validation Checkpoint

  **Prerequisites**: All Phase 11 tasks complete
  
  **Acceptance Criteria**:
  - Complete data flow works from UI to Elasticsearch
  - All integration points verified
  - Documentation complete and accurate
  - E2E tests passing
  
  - [ ] 39.1 Perform manual end-to-end test
    - Start all services via docker-compose
    - Open UI at <http://localhost:3000>
    - Navigate to Data Ingestion page
    - Submit test event
    - Verify in Flow Monitor dashboard
    - Search for entity in Search page
    - Verify entity appears in results
  
  - [ ] 39.2 Run automated E2E test suite
    - Execute: `./scripts/test-e2e-flow.sh`
    - Verify all checks pass
    - Review test output for any warnings
  
  - [ ] 39.3 Verify in Kibana logs
    - Open Kibana at <http://localhost:5601>
    - Search for trace_id from test event
    - Verify logs from all services (Ingest, Resolve, Search)
    - Confirm no errors in pipeline
  
  - [ ] 39.4 Performance validation
    - Measure end-to-end latency (UI submit → Elasticsearch indexed)
    - Target: < 30 seconds for complete flow
    - Document actual timings
  
  - [ ] 39.5 Create validation report
    - Document: All services tested, Integration points verified, Performance metrics, Known issues (if any)
    - File: `docs/validation/e2e-flow-validation-report.md`
  
  Ask user if any questions arise before proceeding.

---

## Summary

**Total Tasks**: 39 major tasks with 150+ sub-tasks
**Estimated Completion**: 8-10 weeks for full implementation
**Current Status**: Infrastructure and core services complete, E2E flow validation in progress

**Next Steps**:

1. Implement Data Ingestion UI (Task 35)
2. Implement Flow Monitor Dashboard (Task 36)
3. Create E2E integration tests (Task 37)
4. Complete documentation (Task 38)
5. Final validation (Task 39)
