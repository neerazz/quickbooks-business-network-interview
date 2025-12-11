# Requirements Document

## Introduction

The QuickBooks Business Network system enables users to efficiently map, visualize, and navigate business relationship graphs. The system identifies and manages vendor-client connections, allowing businesses to understand their network of interactions, discover relationships, and expand their business connections within a network of up to 1 million businesses.

## Glossary

- **Business Network System**: The QuickBooks subsystem responsible for managing and visualizing business relationship graphs
- **Business Entity**: A company or organization registered within the QuickBooks Business Network
- **Relationship**: A vendor-client connection between two Business Entities, weighted by transaction volume
- **Direct Relationship**: A first-degree connection where one Business Entity is directly a vendor or client of another
- **Indirect Relationship**: A connection between Business Entities through one or more intermediary businesses
- **Network Map**: A visual representation of Business Entities and their Relationships
- **Transaction Volume**: The aggregate monetary value of transactions between two connected Business Entities
- **Relationship Weight**: A numerical value representing the Transaction Volume between two Business Entities
- **Entity Resolution**: The process of determining whether two Business Entity references refer to the same real-world business
- **False Positive**: When the Entity Resolution system incorrectly identifies two different businesses as the same entity
- **False Negative**: When the Entity Resolution system fails to identify two references to the same business as matching
- **Training Data**: Annotated examples of Business Entity matches and non-matches used to improve Entity Resolution accuracy
- **Annotation Service**: A system component that processes and labels data for machine learning model training
- **SLA Metric**: Service Level Agreement measurement tracking system performance against defined targets
- **QuickBooks Event**: Any activity or transaction that occurs in the QuickBooks system (invoices, payments, vendor additions, etc.)
- **SQL Database**: A PostgreSQL relational database storing raw QuickBooks Events
- **Kafka Topic**: A message stream for specific event types enabling asynchronous processing
- **Ingest Flow**: The process of receiving QuickBooks Events, storing them in PostgreSQL, and publishing to Kafka Topics
- **Resolve Flow**: The process of transforming messy business data into clean Canonical Entities using AI-powered Entity Resolution
- **Canonical Entity**: A clean, deduplicated representation of a Business Entity with standardized attributes
- **Graph Database**: A Neo4j database optimized for storing Canonical Entities as nodes and their Relationships as edges
- **Elasticsearch**: A search engine database optimized for fast text-based queries and entity lookup
- **AI Service**: A machine learning service that performs Entity Resolution on messy business data
- **Training Consumer**: A Kafka consumer that collects data for training the Entity Resolution model
- **Search API**: An API endpoint that queries Elasticsearch and retrieves connected nodes from Neo4j
- **Data Partitioning**: The strategy of dividing data across multiple database instances based on partition keys
- **Sharding**: The distribution of data across multiple physical servers or availability zones for scalability and fault tolerance
- **Availability Zone**: An isolated location within a cloud region providing redundancy and fault tolerance
- **Vector Database**: A Pinecone/Milvus database that stores vector embeddings for semantic search and similarity matching
- **Graph RAG**: Retrieval-Augmented Generation using graph and vector databases to answer natural language queries about business networks
- **Vector Embedding**: A 768-dimensional numerical representation of business entity attributes and relationships
- **Confidence Threshold**: The minimum match probability (0.85) below which entity pairs are flagged for manual review
- **Valid Characters**: Alphanumeric characters (a-z, A-Z, 0-9), spaces, hyphens (-), apostrophes ('), periods (.), and ampersands (&)
- **Trace ID**: A unique identifier (UUID) that tracks a request across all microservices in a distributed system
- **Span ID**: A unique identifier for a single unit of work within a trace
- **ELK Stack**: Elasticsearch, Logstash, and Kibana - a centralized logging and visualization platform
- **Logstash**: A log aggregation and processing pipeline that ingests logs from multiple sources
- **Kibana**: A web-based visualization and analytics platform for Elasticsearch data
- **Spring Cloud Sleuth**: A distributed tracing library that automatically adds trace and span IDs to logs
- **Structured Logging**: Log format using JSON with consistent fields for machine parsing and analysis

## Requirements

### Requirement 1: Network Map Visualization

**User Story:** As a business owner, I want to view a map of my business's network, so that I can visualize all my vendors and clients and understand my business ecosystem.

#### Acceptance Criteria

1. WHEN a user requests their network map THEN the Business Network System SHALL retrieve all Direct Relationships for that Business Entity from Neo4j within 500ms
2. WHEN displaying the network map THEN the Business Network System SHALL render all connected Business Entities with their relationship types (VENDOR or CLIENT)
3. WHEN rendering relationships THEN the Business Network System SHALL display the Relationship Weight as a decimal value with 2 decimal places
4. WHEN the network map loads THEN the Business Network System SHALL complete the operation within 2 seconds for networks up to 100 Direct Relationships at the 95th percentile
5. WHEN multiple users request network maps simultaneously THEN the Business Network System SHALL handle up to 1,000 concurrent requests without response time degradation exceeding 20%

### Requirement 2: Business and Relationship Search

**User Story:** As a business owner, I want to search for businesses and relationships in multiple ways, so that I can discover connections and business opportunities through various search strategies.

#### Acceptance Criteria

1. WHEN a user searches for a Business Entity by name THEN the Business Network System SHALL return all matching Business Entities ranked by relevance score (0.0-1.0) in descending order
2. WHEN a user searches by relationship type (VENDOR or CLIENT) THEN the Business Network System SHALL return only Business Entities matching the specified relationship type
3. WHEN a user selects a Business Entity from search results THEN the Business Network System SHALL display all Direct Relationships for that Business Entity within 200ms
4. WHEN viewing a Business Entity THEN the Business Network System SHALL compute and display Indirect Relationships up to exactly three degrees of separation (no more, no less)
5. WHEN search queries are submitted THEN the Business Network System SHALL return results within 500 milliseconds at the 95th percentile using Elasticsearch

### Requirement 3: Add Business Relationships

**User Story:** As a business owner, I want to add a new vendor or client to my network, so that I can expand my business relationships and keep my network current.

#### Acceptance Criteria

1. WHEN a user adds a new Business Entity THEN the Business Network System SHALL validate that the entity name is non-empty (minimum 1 character) and contains only Valid Characters (alphanumeric, spaces, hyphens, apostrophes, periods, ampersands)
2. WHEN a user specifies a new Business Entity that does not exist THEN the Business Network System SHALL create the Business Entity with a unique UUID identifier in Neo4j
3. WHEN a user specifies a Business Entity that already exists (matched via Entity Resolution with confidence >= 0.85) THEN the Business Network System SHALL create a Relationship to the existing Business Entity without creating a duplicate
4. WHEN creating a Relationship THEN the Business Network System SHALL require the user to specify exactly one relationship type: VENDOR or CLIENT
5. WHEN a Relationship is created THEN the Business Network System SHALL initialize the Relationship Weight to exactly 0.00

### Requirement 4: Entity Resolution and Deduplication

**User Story:** As a business owner, I want the system to automatically handle duplicate business entities intelligently, so that my network remains clean without manual intervention.

#### Acceptance Criteria

1. WHEN a Ledger Entry references a Business Entity THEN the Entity Resolution system SHALL automatically identify if the entity already exists in the network using the four-stage pipeline (Standardize, Block, Compare, Classify)
2. WHEN Entity Resolution detects a match with confidence >= 0.85 THEN the Business Network System SHALL merge the entity with the existing Canonical Entity automatically
3. WHEN merging entities THEN the Business Network System SHALL consolidate all Relationships and sum all Transaction Volumes under the canonical entity, preserving no orphaned relationships
4. WHEN Entity Resolution confidence is between 0.50 and 0.85 THEN the Business Network System SHALL flag the entity pair for user review with the confidence score displayed
5. WHEN Entity Resolution confidence is below 0.50 THEN the Business Network System SHALL treat the entity as a new unique Business Entity

### Requirement 5: System Availability and Performance

**User Story:** As a system administrator, I want the system to maintain high availability and performance, so that users can access their business networks reliably at any time.

#### Acceptance Criteria

1. WHEN the system experiences high query load (up to 10 million searches per month) THEN the Business Network System SHALL maintain response times at the 95th percentile under 1 second
2. WHEN frequently accessed relationships are queried (top 20% by access frequency) THEN the Business Network System SHALL serve results from Redis cache with TTL of 5 minutes
3. WHEN cache entries expire THEN the Business Network System SHALL refresh cached data asynchronously without blocking user requests
4. WHEN the system processes requests THEN the Business Network System SHALL distribute load across multiple service instances using round-robin load balancing
5. WHEN a component fails THEN the Business Network System SHALL continue serving requests using redundant components with automatic failover within 30 seconds

### Requirement 6: Transaction Volume Updates

**User Story:** As a business owner, I want to update transaction volumes between my business and partners, so that relationship weights accurately reflect current business activity.

#### Acceptance Criteria

1. WHEN transaction data is recorded THEN the Business Network System SHALL recalculate the Relationship Weight by summing all transaction amounts between the two Business Entities
2. WHEN a Relationship Weight changes THEN the Business Network System SHALL update the stored weight value atomically using database transactions
3. WHEN displaying relationships THEN the Business Network System SHALL show the most recent Relationship Weight values (eventually consistent within 5 seconds)
4. WHEN multiple transactions affect the same Relationship within the same second THEN the Business Network System SHALL aggregate Transaction Volumes correctly using idempotent operations
5. WHEN weight updates occur THEN the Business Network System SHALL complete the update within 100 milliseconds at the 95th percentile

### Requirement 7: Relationship Removal

**User Story:** As a business owner, I want to remove relationships from my network, so that I can maintain an accurate representation of my current business connections.

#### Acceptance Criteria

1. WHEN a user requests to remove a Relationship THEN the Business Network System SHALL delete the edge between the two Business Entity nodes in Neo4j
2. WHEN a Relationship is removed THEN the Business Network System SHALL preserve both Business Entity nodes in Neo4j (soft delete of edge only)
3. WHEN a Relationship is deleted THEN the Business Network System SHALL invalidate cached network maps containing either Business Entity within 1 second
4. WHEN removing a Relationship THEN the Business Network System SHALL require user confirmation via a confirmation dialog before deletion
5. WHEN a Relationship is removed THEN the Business Network System SHALL maintain referential integrity by removing the relationship from both entities' adjacency lists

### Requirement 8: Network Data Export

**User Story:** As a business owner, I want to export my network data, so that I can analyze relationships using external tools or share information with stakeholders.

#### Acceptance Criteria

1. WHEN a user requests a network export THEN the Business Network System SHALL generate a file containing all Business Entities and Relationships accessible to that user
2. WHEN exporting data THEN the Business Network System SHALL include: entity_id, entity_name, entity_category, relationship_id, relationship_type, relationship_weight, source_entity_id, target_entity_id
3. WHEN generating exports THEN the Business Network System SHALL support JSON format (application/json) and CSV format (text/csv) with proper escaping of special characters
4. WHEN an export is requested THEN the Business Network System SHALL complete the operation within 5 seconds for networks up to 100 Direct Relationships at the 95th percentile
5. WHEN exporting data THEN the Business Network System SHALL include only relationships where the requesting user's Business Entity is either the source or target

### Requirement 9: Test Data Generation

**User Story:** As a developer, I want a data generator that creates realistic QuickBooks events, so that I can populate the POC application with pre-generated transaction data.

#### Acceptance Criteria

1. WHEN the data generator executes THEN the Business Network System SHALL create exactly 10,000 unique business identities with realistic but messy names, addresses, and categories
2. WHEN generating event data THEN the Business Network System SHALL create exactly 1,000,000 QuickBooks Events representing transaction types: INVOICE, PAYMENT, VENDOR_ADD, CUSTOMER_ADD
3. WHEN creating QuickBooks Events THEN the Business Network System SHALL include messy business data with variations: 30% with typos, 40% with abbreviations, 50% with inconsistent capitalization
4. WHEN generating events THEN the Business Network System SHALL distribute events across businesses following a power-law distribution (80/20 rule)
5. WHEN the data generator completes THEN the Business Network System SHALL store all QuickBooks Events in PostgreSQL with status=PENDING ready for ingestion

### Requirement 10: POC User Simulation

**User Story:** As a developer testing the POC, I want to view all users on a home page and select any user to simulate their session, so that I can test the system from different user perspectives without authentication.

#### Acceptance Criteria

1. WHEN the POC home page loads THEN the Business Network System SHALL display a paginated list (20 per page) of all Business Entities sorted alphabetically by name
2. WHEN a user is selected from the list THEN the Business Network System SHALL set a session cookie (simulated_user_id) and navigate to that Business Entity's dashboard
3. WHEN viewing a Business Entity's dashboard THEN the Business Network System SHALL display the interface as if that Business Entity is the authenticated user
4. WHEN simulating a user session THEN the Business Network System SHALL enable all CRUD functionality for that Business Entity's relationships
5. WHEN switching between users THEN the Business Network System SHALL clear the previous session and establish a new session within 100ms

### Requirement 11: SLA Metrics Tracking

**User Story:** As a system administrator, I want to track SLA metrics for all API calls, so that I can monitor system performance and ensure service level agreements are met.

#### Acceptance Criteria

1. WHEN an API call is made THEN the Business Network System SHALL record the response time in milliseconds with microsecond precision
2. WHEN calculating SLA metrics THEN the Business Network System SHALL compute percentile statistics: p50, p75, p90, p95, p99 for response times per endpoint
3. WHEN API calls complete THEN the Business Network System SHALL track success rate (2xx responses) and error rates (4xx, 5xx) by endpoint per minute
4. WHEN viewing SLA metrics THEN the Business Network System SHALL display current performance against defined targets: p95 < 1000ms, error rate < 1%, availability > 99.9%
5. WHEN SLA thresholds are breached for 5 consecutive minutes THEN the Business Network System SHALL generate alerts via configured notification channels (webhook, email)

### Requirement 12: Training Data Annotation

**User Story:** As a data scientist, I want a training data annotation service that processes business entity data, so that I can create labeled datasets to improve the entity resolution model.

#### Acceptance Criteria

1. WHEN the Annotation Service processes entity pairs THEN the Business Network System SHALL extract features: standardized_name, address_components, category, legal_entity_type
2. WHEN annotating data THEN the Annotation Service SHALL label entity pairs as MATCH (same entity), NON_MATCH (different entities), or UNCERTAIN (requires review)
3. WHEN generating Training Data THEN the Annotation Service SHALL create balanced datasets with 40% positive examples (matches), 40% negative examples (non-matches), 20% uncertain
4. WHEN Training Data is created THEN the Annotation Service SHALL store annotations in PostgreSQL in format: {entity_a_id, entity_b_id, label, feature_vector, annotator, timestamp}
5. WHEN the annotation process completes a batch THEN the Annotation Service SHALL validate data quality: no duplicate pairs, all required fields populated, labels in valid set

### Requirement 13: User Feedback for Entity Resolution

**User Story:** As a business owner, I want to report when the system incorrectly matches or fails to match business entities, so that the system can learn from mistakes and improve accuracy.

#### Acceptance Criteria

1. WHEN the system suggests a Business Entity match THEN the Business Network System SHALL display a "Report Incorrect Match" button with the entity pair details
2. WHEN the system fails to suggest a match THEN the Business Network System SHALL provide a "Report Missing Match" link in the search results interface
3. WHEN a user reports a False Positive THEN the Business Network System SHALL record: entity_a_id, entity_b_id, feedback_type=FALSE_POSITIVE, user_id, timestamp, optional_comment
4. WHEN a user reports a False Negative THEN the Business Network System SHALL record: entity_a_id, entity_b_id (user-specified), feedback_type=FALSE_NEGATIVE, user_id, timestamp, optional_comment
5. WHEN feedback is submitted THEN the Business Network System SHALL acknowledge receipt within 200ms and queue for processing

### Requirement 14: Continuous Model Improvement

**User Story:** As a data scientist, I want user feedback to be incorporated into model training, so that the entity resolution system continuously improves based on real-world corrections.

#### Acceptance Criteria

1. WHEN user feedback is collected THEN the Business Network System SHALL add reported cases to the training_data table with source=USER_FEEDBACK
2. WHEN preparing training datasets THEN the Business Network System SHALL weight user-reported corrections 3x higher than automated annotations
3. WHEN model retraining is triggered (weekly or when 1000+ new feedback items exist) THEN the Business Network System SHALL include all validated user feedback in the training set
4. WHEN feedback conflicts with existing Training Data THEN the Business Network System SHALL mark the existing annotation as superseded and use user feedback as ground truth
5. WHEN Training Data is updated THEN the Business Network System SHALL track metadata: source (AUTOMATED, USER_FEEDBACK), confidence_level (0.0-1.0), created_at, updated_at

### Requirement 15: Entity Resolution Accuracy

**User Story:** As a system architect, I want an Entity Resolution Service that achieves 99.99% accuracy through a multi-stage process, so that business entities are correctly identified and deduplicated even with significant name variations.

#### Acceptance Criteria

1. WHEN the Entity Resolution Service processes business data THEN the system SHALL achieve >= 99.99% accuracy measured as (True Positives + True Negatives) / Total on the holdout test set
2. WHEN comparing entities THEN the Entity Resolution Service SHALL correctly identify case variations ("ACME Corp", "acme corp", "Acme CORP") as the same entity
3. WHEN comparing entities THEN the Entity Resolution Service SHALL correctly identify abbreviation variations ("ACME Corp", "ACME Corporation", "A.C.M.E. Corp.") as the same entity
4. WHEN comparing entities THEN the Entity Resolution Service SHALL correctly differentiate legal entity types ("ACME LLC" vs "ACME Corp" vs "ACME Inc") as different entities
5. WHEN accuracy metrics are calculated monthly THEN the Business Network System SHALL track and report both False Positive Rate (<0.01%) and False Negative Rate (<0.01%)

### Requirement 16: Ingest Flow

**User Story:** As a system architect, I want an Ingest Flow that stores QuickBooks Events in PostgreSQL and publishes them to Kafka Topics, so that events can be processed by multiple downstream consumers.

#### Acceptance Criteria

1. WHEN a QuickBooks Event is received THEN the Ingest Flow SHALL store the complete event data in PostgreSQL with columns: id, event_type, source_business_raw, target_business_raw, amount, metadata, event_timestamp, ingested_at, status
2. WHEN storing an event THEN the Ingest Flow SHALL publish the event to the appropriate Kafka Topic based on event type
3. WHEN publishing to Kafka THEN the Ingest Flow SHALL route events to topics: quickbooks.events.invoice, quickbooks.events.payment, quickbooks.events.vendor, quickbooks.events.customer
4. WHEN Kafka publishing fails THEN the Ingest Flow SHALL retry with exponential backoff (initial=100ms, max=30s, multiplier=2) for up to 5 attempts
5. WHEN the Ingest Flow processes an event THEN the Business Network System SHALL ensure both PostgreSQL storage and Kafka publishing succeed atomically using the transactional outbox pattern

### Requirement 17: Resolve Flow

**User Story:** As a system architect, I want a Resolve Flow that transforms messy business data into Canonical Entities using an AI Service, so that the graph contains clean, deduplicated business information.

#### Acceptance Criteria

1. WHEN the Resolve Flow consumes a QuickBooks Event from Kafka THEN the system SHALL extract business entity information: source_business_raw, target_business_raw, amount
2. WHEN messy business data is extracted THEN the Resolve Flow SHALL send the data to the Entity Resolution Service via HTTP POST to /api/v1/entity-resolution/resolve
3. WHEN the Entity Resolution Service returns a Canonical Entity THEN the system SHALL receive: canonical_name, standardized_address, category, legal_entity_type, confidence_score, aliases[]
4. WHEN a Canonical Entity is created THEN the Resolve Flow SHALL check Neo4j for existing entities using blocking keys (name_prefix + state)
5. WHEN Entity Resolution identifies a match (confidence >= 0.85) THEN the Resolve Flow SHALL merge: update existing node, add new aliases, sum transaction volumes, consolidate relationships

### Requirement 18: Graph Database Persistence

**User Story:** As a system architect, I want Canonical Entities persisted in Neo4j as clean nodes, so that the business network can be queried efficiently and reliably at scale.

#### Acceptance Criteria

1. WHEN a Canonical Entity is resolved THEN the Business Network System SHALL create or update a node in Neo4j with label :Business
2. WHEN storing nodes THEN Neo4j SHALL include properties: id (UUID), canonical_name, raw_name, category, address_street, address_city, address_state, address_zip, aliases[], created_at, updated_at
3. WHEN QuickBooks Events indicate relationships THEN the Business Network System SHALL create edges with type :VENDOR_OF or :CLIENT_OF between Business nodes
4. WHEN creating edges THEN Neo4j SHALL store properties: id (UUID), transaction_volume (decimal), weight (decimal), created_at, last_transaction_at
5. WHEN Neo4j scales beyond 1 million nodes THEN the system SHALL maintain query performance of <500ms for 3-hop traversals via proper indexing on id, canonical_name, and state

### Requirement 19: Elasticsearch Indexing

**User Story:** As a system architect, I want Canonical Entities indexed in Elasticsearch, so that the Search API can quickly locate entities before fetching their network connections.

#### Acceptance Criteria

1. WHEN a Canonical Entity is persisted in Neo4j THEN the Business Network System SHALL create a corresponding document in Elasticsearch index 'business_entities'
2. WHEN indexing in Elasticsearch THEN the system SHALL include fields: id (keyword), canonical_name (text with ngram analyzer), category (keyword), state (keyword), aliases (text[]), created_at (date)
3. WHEN entity data is updated in Neo4j THEN the Business Network System SHALL synchronize changes to Elasticsearch within 5 seconds (eventual consistency)
4. WHEN Elasticsearch indexing fails THEN the system SHALL retry 3 times with exponential backoff and log to dead letter queue on final failure
5. WHEN searching entities THEN Elasticsearch SHALL return results within 100 milliseconds for typical queries at the 95th percentile

### Requirement 20: Search API with Graph Traversal

**User Story:** As a business owner, I want a Search API that finds entities and retrieves their connected nodes up to 3 levels deep, so that I can explore my business network comprehensively.

#### Acceptance Criteria

1. WHEN a user submits a search query THEN the Search API SHALL query Elasticsearch to locate matching Canonical Entities and return top 20 results by relevance
2. WHEN Elasticsearch returns matching entities THEN the Search API SHALL retrieve the selected entity's node from Neo4j by id
3. WHEN fetching network connections THEN the Search API SHALL execute a Cypher query to traverse Neo4j and retrieve all connected nodes up to exactly 3 levels deep
4. WHEN filtering connections THEN the Search API SHALL support query parameters: type (VENDOR|CLIENT), min_volume (decimal), max_volume (decimal), category (string), state (string)
5. WHEN returning results THEN the Search API SHALL complete the entire operation (Elasticsearch + Neo4j traversal) within 1 second at the 95th percentile for networks with up to 100 direct connections per node

### Requirement 21: Training Consumer

**User Story:** As a data scientist, I want a Training Consumer that reads QuickBooks Events from Kafka and generates training data, so that the Entity Resolution model continuously improves.

#### Acceptance Criteria

1. WHEN the Training Consumer reads a QuickBooks Event from Kafka THEN the system SHALL extract all business entity references (source_business_raw, target_business_raw) for training data generation
2. WHEN processing events THEN the Training Consumer SHALL create entity pair examples pairing the raw entity with its resolved Canonical Entity
3. WHEN generating training data THEN the Training Consumer SHALL include 50% positive examples (confirmed matches) and 50% negative examples (confirmed non-matches from different blocks)
4. WHEN user feedback exists for an entity pair THEN the Training Consumer SHALL incorporate the feedback label as ground truth, overriding automated labels
5. WHEN training data is generated THEN the Training Consumer SHALL store in PostgreSQL table training_examples: id, entity_a_raw, entity_b_raw, label, feature_vector (JSON), source, confidence, created_at

### Requirement 22: Entity Resolution Four-Stage Pipeline

**User Story:** As a data engineer, I want the Entity Resolution Service to follow a four-stage process (Standardize, Block, Compare, Classify), so that entity matching is both accurate and computationally efficient.

#### Acceptance Criteria

1. WHEN the Standardize stage processes business data THEN the Entity Resolution Service SHALL output: standardized_name, legal_entity_type, standardized_address, category, tokens[]
2. WHEN the Block stage executes THEN the Entity Resolution Service SHALL group entities by blocking keys and reduce comparison space by >= 95% while maintaining >= 99% recall
3. WHEN the Compare stage processes blocked groups THEN the Entity Resolution Service SHALL compute similarity scores and output feature vectors with 7 features per entity pair
4. WHEN the Classify stage evaluates feature vectors THEN the Entity Resolution Service SHALL output: match_probability (0.0-1.0), is_match (boolean), confidence (0.0-1.0), requires_review (boolean)
5. WHEN all four stages complete THEN the Entity Resolution Service SHALL return a complete EntityResolutionResult within 100ms at the 95th percentile

### Requirement 23: Standardize Stage

**User Story:** As a data engineer, I want the Standardize stage to clean and normalize business data, so that subsequent matching stages work with consistent, high-quality input.

#### Acceptance Criteria

1. WHEN standardizing business names THEN the Entity Resolution Service SHALL: convert to lowercase, collapse multiple whitespace to single space, trim leading/trailing whitespace
2. WHEN standardizing names THEN the Entity Resolution Service SHALL expand abbreviations using mapping: Corp→Corporation, Inc→Incorporated, Ltd→Limited, LLC→Limited Liability Company, Co→Company
3. WHEN standardizing addresses THEN the Entity Resolution Service SHALL normalize: St→Street, Ave→Avenue, Blvd→Boulevard, Rd→Road, Dr→Drive, N→North, S→South, E→East, W→West, Apt→Apartment, Ste→Suite
4. WHEN standardizing text THEN the Entity Resolution Service SHALL remove characters: . , ; : ! ? ( ) [ ] { } but preserve apostrophes (') and hyphens (-)
5. WHEN standardizing any input THEN the Entity Resolution Service SHALL preserve the original raw value alongside the standardized output for audit and debugging purposes
5. WHEN standardization completes THEN the Entity Resolution Service SHALL return both: original_raw (unchanged) and standardized fields (cleaned)

### Requirement 24: Block Stage

**User Story:** As a data engineer, I want the Block stage to group likely matches efficiently, so that the system avoids comparing every entity pair while maintaining high recall.

#### Acceptance Criteria

1. WHEN creating blocks THEN the Entity Resolution Service SHALL generate blocking keys: first_3_chars_of_name (e.g., "acm"), state_code (e.g., "CA"), first_word_plus_state (e.g., "acme-CA")
2. WHEN an entity is processed THEN the Entity Resolution Service SHALL assign it to all blocks matching its blocking keys (multi-block assignment)
3. WHEN blocks are formed THEN the Entity Resolution Service SHALL ensure true matches have >= 99% probability of sharing at least one block (recall >= 99%)
4. WHEN a block size exceeds 1,000 entities THEN the Entity Resolution Service SHALL subdivide using additional criteria: add category to key, then add zip_prefix
5. WHEN blocking completes THEN the Entity Resolution Service SHALL log metrics: total_entities, total_blocks, average_block_size, max_block_size, comparison_reduction_percentage

### Requirement 25: Compare Stage

**User Story:** As a data engineer, I want the Compare stage to compute similarity scores using fuzzy matching algorithms, so that entity pairs can be evaluated for potential matches.

#### Acceptance Criteria

1. WHEN comparing entity names THEN the Entity Resolution Service SHALL compute three string similarity scores: levenshtein_distance (0.0-1.0), jaro_winkler (0.0-1.0), token_sort_ratio (0.0-1.0)
2. WHEN comparing addresses THEN the Entity Resolution Service SHALL calculate address_similarity (0.0-1.0) as weighted average: street (0.4), city (0.3), state (0.2), zip (0.1)
3. WHEN comparing entities THEN the Entity Resolution Service SHALL generate a 7-feature vector: levenshtein_score, jaro_winkler_score, token_similarity, address_similarity, category_match (0 or 1), same_legal_type (0 or 1), same_state (0 or 1)
4. WHEN similarity scores are computed THEN the Entity Resolution Service SHALL normalize all continuous scores to the range [0.0, 1.0] using min-max normalization
5. WHEN the Compare stage completes THEN the Entity Resolution Service SHALL output feature vectors for all entity pairs within each block with no missing features

### Requirement 26: Classify Stage

**User Story:** As a data scientist, I want the Classify stage to use an AI model for match classification, so that the system learns from training data and improves accuracy over time.

#### Acceptance Criteria

1. WHEN the Classify stage receives feature vectors THEN the Entity Resolution Service SHALL input them to a trained Random Forest or XGBoost model
2. WHEN the AI model processes features THEN the Entity Resolution Service SHALL output a match_probability score between 0.0 and 1.0 (inclusive)
3. WHEN match_probability >= 0.85 THEN the Entity Resolution Service SHALL classify as is_match=true with requires_review=false
4. WHEN match_probability is between 0.50 and 0.85 THEN the Entity Resolution Service SHALL classify as is_match=false with requires_review=true
5. WHEN the AI model is retrained THEN the Entity Resolution Service SHALL use a holdout test set (20% of data) to validate accuracy >= 99.99% before deployment

### Requirement 27: Developer Setup Documentation

**User Story:** As a developer, I want comprehensive documentation on system setup and operations, so that I can quickly clone, configure, and run the application without prior knowledge.

#### Acceptance Criteria

1. WHEN a developer accesses the repository THEN the Business Network System SHALL provide README.md with: git clone command, prerequisite list (Java 17, Node 18, Docker), and dependency installation steps
2. WHEN setting up services THEN the documentation SHALL include: docker-compose up command, service startup order, health check verification commands for each service
3. WHEN configuring databases THEN the documentation SHALL specify: all JDBC/connection URLs, port numbers (PostgreSQL: 5432, Neo4j: 7687, Elasticsearch: 9200, Kafka: 9092, Redis: 6379)
4. WHEN managing secrets THEN the documentation SHALL list all required environment variables in .env.example with descriptions and example values (never real secrets)
5. WHEN troubleshooting THEN the documentation SHALL include a troubleshooting section with: 10 most common errors, their causes, and resolution steps

### Requirement 28: User Guide Documentation

**User Story:** As a new user, I want high-level application usage documentation, so that I can understand the system's capabilities and how to perform key tasks.

#### Acceptance Criteria

1. WHEN a user accesses the documentation THEN the Business Network System SHALL provide docs/user-guide.md with: system overview, feature list, and 5 primary use cases
2. WHEN learning to use the system THEN the documentation SHALL include step-by-step guides with screenshots for: viewing network map, searching entities, adding relationships, exporting data, using Graph RAG
3. WHEN viewing documentation THEN the system SHALL provide 10+ example search queries with expected results and explanation of relevance ranking
4. WHEN understanding the architecture THEN the documentation SHALL include Mermaid diagrams showing: high-level architecture, data flow, sequence diagrams for key operations
5. WHEN accessing help THEN the documentation SHALL be organized with: table of contents, searchable headings, and cross-references to related sections

### Requirement 29: Repository Structure

**User Story:** As a developer, I want a clear top-level directory structure, so that I can easily locate source code, documentation, configuration, and deployment files.

#### Acceptance Criteria

1. WHEN the repository is structured THEN the Business Network System SHALL organize all Java service source code under src/services/{service-name}/
2. WHEN organizing documentation THEN the Business Network System SHALL place: setup guides in docs/setup/, user guides in docs/user-guide/, API docs in docs/api/, architecture in docs/architecture/
3. WHEN managing CI/CD THEN the Business Network System SHALL store: GitHub Actions workflows in .github/workflows/, PR template in .github/pull_request_template.md
4. WHEN containerizing services THEN the Business Network System SHALL include: docker-compose.yml at root, individual Dockerfiles in each service directory
5. WHEN viewing the repository THEN the Business Network System SHALL provide README.md at root with: project overview (500+ words), quick start (5 steps), architecture diagram, service list with descriptions

### Requirement 30: Automated Database Population

**User Story:** As a developer, I want automated database population on first startup, so that the system has realistic data available immediately without manual data loading.

#### Acceptance Criteria

1. WHEN the system starts for the first time THEN the Business Network System SHALL detect empty databases by checking: PostgreSQL quickbooks_events table count = 0, Neo4j Business node count = 0
2. WHEN populating databases THEN the system SHALL execute the Data Generator to create 1,000,000 QuickBooks Events in PostgreSQL (approximately 10 minutes)
3. WHEN data generation completes THEN the system SHALL trigger the Ingest Flow to process all PENDING events and publish to Kafka topics
4. WHEN Kafka topics receive events THEN the Resolve Flow SHALL consume and process events, populating Neo4j (nodes, edges) and Elasticsearch (documents)
5. WHEN automated population completes THEN the Business Network System SHALL log: "Database population complete. Businesses: {count}, Events: {count}, Relationships: {count}" and set status flag populated=true

### Requirement 31: PostgreSQL Data Partitioning

**User Story:** As a system architect, I want data partitioning and sharding strategies for multi-availability zone deployment, so that the system achieves high availability, fault tolerance, and horizontal scalability.

#### Acceptance Criteria

1. WHEN storing data in PostgreSQL THEN the Business Network System SHALL partition quickbooks_events table by event_timestamp using monthly range partitions
2. WHEN distributing data THEN the Business Network System SHALL configure PostgreSQL with 1 primary and 2 read replicas across 3 Availability Zones
3. WHEN a primary fails THEN the Business Network System SHALL promote a replica to primary within 30 seconds using automatic failover
4. WHEN data is written THEN the Business Network System SHALL use synchronous replication to at least 1 replica before acknowledging writes
5. WHEN query load increases THEN the Business Network System SHALL route read queries to replicas using connection pooling (PgBouncer) with round-robin distribution

### Requirement 32: Neo4j Graph Database Sharding

**User Story:** As a system architect, I want the Graph Database to support sharding across Availability Zones, so that graph queries remain performant and available even during zone failures.

#### Acceptance Criteria

1. WHEN storing Canonical Entities THEN Neo4j SHALL use Neo4j Fabric to federate queries across 3 database instances (1 per AZ)
2. WHEN creating relationships THEN Neo4j SHALL co-locate entities that frequently query together based on state (geographic locality)
3. WHEN a shard becomes unavailable THEN Neo4j SHALL route queries to replica shards using Causal Clustering with automatic leader election
4. WHEN traversing the graph across shards THEN Neo4j SHALL execute distributed queries with fan-out and merge, completing 3-hop traversals in <1 second
5. WHEN rebalancing shards THEN Neo4j SHALL redistribute data online without service interruption using background replication

### Requirement 33: Elasticsearch and Kafka Multi-AZ Deployment

**User Story:** As a system architect, I want Elasticsearch and Kafka to be deployed across multiple Availability Zones, so that search and event streaming remain available during zone failures.

#### Acceptance Criteria

1. WHEN deploying Elasticsearch THEN the Business Network System SHALL configure 3 master-eligible nodes (1 per AZ) and 6 data nodes (2 per AZ)
2. WHEN an Elasticsearch node fails THEN the system SHALL continue serving queries using replica shards with 0 downtime (replication factor = 2)
3. WHEN deploying Kafka THEN the Business Network System SHALL configure 3 brokers (1 per AZ) with topic replication factor = 3
4. WHEN a Kafka broker fails THEN the system SHALL continue producing and consuming using in-sync replicas with automatic leader election
5. WHEN network partitions occur THEN the Business Network System SHALL maintain consistency using: Elasticsearch minimum_master_nodes=2, Kafka min.insync.replicas=2

### Requirement 34: Graph RAG Natural Language Queries

**User Story:** As a business owner, I want to ask natural language questions about my business network using Graph RAG, so that I can discover insights without learning complex query syntax.

#### Acceptance Criteria

1. WHEN a user submits a natural language query THEN the Graph RAG service SHALL parse the query using NLP to extract: intent (search|analyze|recommend), entities[], filters{}, max_depth
2. WHEN processing queries like "flower businesses in Texas connected to me offering discounts" THEN the Graph RAG service SHALL extract: category=flower, state=TX, relationship=connected, attribute=offers_discounts
3. WHEN executing queries THEN the Graph RAG service SHALL first query Vector DB for semantic matches, then query Neo4j for relationship traversal
4. WHEN combining results THEN the Graph RAG service SHALL merge Vector DB results (semantic similarity) with Neo4j results (graph connectivity) ranked by combined score
5. WHEN returning results THEN the Graph RAG service SHALL respond with: results[] (max 20), confidence_score (0.0-1.0), explanation (natural language summary of query interpretation)

### Requirement 35: Vector Database for Semantic Search

**User Story:** As a system architect, I want a Vector Database that stores embeddings of business entities and relationships, so that the Graph RAG service can perform semantic search and similarity matching.

#### Acceptance Criteria

1. WHEN a Canonical Entity is created or updated THEN the Business Network System SHALL generate a 768-dimensional vector embedding using a pre-trained sentence transformer model
2. WHEN storing embeddings THEN the Vector Database (Pinecone/Milvus) SHALL index vectors using HNSW algorithm with M=16, efConstruction=200 for fast approximate nearest neighbor search
3. WHEN the Graph RAG service queries by semantic meaning THEN the Vector Database SHALL return top-K entities (K=100) with cosine similarity >= 0.7
4. WHEN entity attributes are updated THEN the Business Network System SHALL regenerate and upsert the corresponding vector embedding within 10 seconds
5. WHEN the Vector Database scales to 1 million entities THEN the system SHALL maintain <100ms query latency for similarity searches at the 95th percentile

### Requirement 36: Graph RAG Query Execution

**User Story:** As a business owner, I want the Graph RAG service to query both vector and graph databases, so that I can find businesses by semantic similarity and then explore their network connections.

#### Acceptance Criteria

1. WHEN the Graph RAG service receives a query THEN the system SHALL first query the Vector Database to find top-100 semantically relevant entities
2. WHEN vector search returns candidate entities THEN the Graph RAG service SHALL extract their Neo4j node IDs from the embedding metadata
3. WHEN node IDs are obtained THEN the Graph RAG service SHALL execute a Cypher query to traverse relationships up to the specified depth (default=2, max=3)
4. WHEN filtering by attributes THEN the Graph RAG service SHALL check entity properties in Neo4j: offers_discounts (boolean), min_volume (decimal), relationship_type (string)
5. WHEN combining results THEN the Graph RAG service SHALL rank by: (0.6 *semantic_similarity) + (0.4* graph_connectivity_score) where connectivity = 1 / (hop_distance + 1)

### Requirement 37: Graph RAG Context Understanding

**User Story:** As a business owner, I want Graph RAG queries to understand business context like "connected to me" and "offers discounts", so that results are relevant to my specific business situation.

#### Acceptance Criteria

1. WHEN a query includes "connected to me" or "my network" THEN the Graph RAG service SHALL resolve "me" to the authenticated user's Business Entity ID from the session
2. WHEN a query specifies relationship depth ("directly connected", "2 hops away", "within 3 degrees") THEN the Graph RAG service SHALL set max_depth accordingly: direct=1, 2 hops=2, 3 degrees=3
3. WHEN a query mentions attributes ("offers discounts", "high volume", "in Texas") THEN the Graph RAG service SHALL convert to Neo4j property filters: offers_discounts=true, volume>=threshold, state='TX'
4. WHEN combining multiple filters THEN the Graph RAG service SHALL apply all conditions using AND logic by default, OR logic when "or" is explicitly used
5. WHEN no results match all criteria THEN the Graph RAG service SHALL return: results=[], suggestions=["Try removing the discount filter", "Expand search to neighboring states"], explanation="No flower businesses in TX with discounts found within 3 hops"

### Requirement 38: Graph RAG REST API

**User Story:** As a developer, I want the Graph RAG service to expose a REST API for AI assistant interactions, so that the system remains simple without requiring WebSocket connections.

#### Acceptance Criteria

1. WHEN a user interacts with the AI assistant THEN the Business Network System SHALL use REST API endpoint POST /api/v1/graph-rag/query
2. WHEN the Graph RAG service receives a REST request THEN the system SHALL process the query synchronously and return complete results in a single HTTP response
3. WHEN API responses are returned THEN the Graph RAG service SHALL include: results[], confidence_score, suggestions[], explanation, query_interpretation{}
4. WHEN the REST API is called THEN the Business Network System SHALL authenticate using Bearer token and identify the requesting Business Entity from the token claims
5. WHEN designing the API THEN the Business Network System SHALL NOT implement WebSocket endpoints for Graph RAG functionality (REST only)

### Requirement 39: Documentation Maintenance

**User Story:** As a developer, I want documentation to be updated every time code is implemented, so that documentation always reflects the current state of the system.

#### Acceptance Criteria

1. WHEN code is implemented or modified THEN the developer SHALL update corresponding documentation before merging the PR
2. WHEN APIs are added or changed THEN the developer SHALL update docs/api/openapi.yaml with new endpoints, parameters, request/response schemas
3. WHEN architecture changes THEN the developer SHALL update docs/architecture/overview.md and regenerate affected Mermaid diagrams
4. WHEN configuration changes THEN the developer SHALL update docs/setup/getting-started.md and .env.example with new configuration requirements
5. WHEN a PR is submitted THEN the CI pipeline SHALL verify documentation was modified if source code in src/ was modified (documentation-required check)

### Requirement 40: Architectural Diagrams

**User Story:** As a developer, I want comprehensive architectural diagrams maintained up-to-date, so that I can understand system structure and data flows at a glance.

#### Acceptance Criteria

1. WHEN the system is documented THEN the Business Network System SHALL include docs/architecture/overview.md with high-level architecture diagram showing all 12 services and 6 data stores
2. WHEN data flows are documented THEN the Business Network System SHALL include Mermaid diagrams for: Ingest Flow (5 steps), Resolve Flow (7 steps), Graph RAG query (6 steps)
3. WHEN services interact THEN the Business Network System SHALL include sequence diagrams for: entity creation, relationship search, Graph RAG query, user feedback submission
4. WHEN diagrams are created THEN the Business Network System SHALL use Mermaid syntax in markdown files for version control and diff visibility
5. WHEN code changes affect architecture THEN the developer SHALL update corresponding diagrams in the same PR before approval

### Requirement 41: Code Quality Standards

**User Story:** As a developer, I want code to follow design patterns and quality standards, so that the codebase is maintainable, extensible, and modular.

#### Acceptance Criteria

1. WHEN implementing new functionality THEN the developer SHALL first check for existing code and extend existing classes/methods rather than duplicating
2. WHEN creating Java classes THEN the developer SHALL apply patterns: Repository pattern for data access, Service pattern for business logic, Factory pattern for object creation, Strategy pattern for algorithms
3. WHEN structuring code THEN the developer SHALL ensure: single responsibility per class, dependencies injected via constructor, interfaces for external integrations
4. WHEN writing code THEN the developer SHALL include structured JSON logging via SLF4J/Logback with fields: timestamp, level, service, correlation_id, message, context{}
5. WHEN designing services THEN the developer SHALL document public APIs with Javadoc: description, @param for each parameter, @return description, @throws for each exception

### Requirement 42: Test Coverage

**User Story:** As a developer, I want comprehensive test coverage with positive and negative test cases, so that code quality is verified and regressions are prevented.

#### Acceptance Criteria

1. WHEN code is created THEN the developer SHALL create corresponding unit tests: minimum 1 positive case (happy path), minimum 2 negative cases (error conditions, edge cases)
2. WHEN tests are written THEN the developer SHALL ensure all tests pass (mvn test, npm test) before code is considered complete
3. WHEN testing Java services THEN the developer SHALL use: JUnit 5 for unit tests, Mockito for mocking, Testcontainers for integration tests with real databases
4. WHEN testing React frontend THEN the developer SHALL use: Jest for unit tests, React Testing Library for component tests, MSW for API mocking
5. WHEN a PR is submitted THEN the CI pipeline SHALL enforce: minimum 80% line coverage, 0 failing tests, no reduction in existing coverage

### Requirement 43: Observability and Reliability

**User Story:** As a system operator, I want the system to have observability, debuggability, and reliability built in, so that I can monitor, troubleshoot, and maintain the system effectively.

#### Acceptance Criteria

1. WHEN services run THEN the Business Network System SHALL expose: /actuator/health (liveness), /actuator/ready (readiness), /actuator/metrics (Prometheus format)
2. WHEN errors occur THEN the Business Network System SHALL log: correlation_id (UUID propagated across services), stack trace, request context, timestamp in ISO8601 format
3. WHEN debugging issues THEN the Business Network System SHALL support log levels configurable per package via environment variable: LOG_LEVEL_{PACKAGE}=DEBUG|INFO|WARN|ERROR
4. WHEN monitoring performance THEN the Business Network System SHALL emit metrics: request_duration_seconds (histogram), request_count (counter), error_count (counter), queue_depth (gauge)
5. WHEN operating the system THEN the Business Network System SHALL provide docs/operations/runbooks.md with procedures for: restart service, scale service, database backup, incident response

### Requirement 44: Clean Code

**User Story:** As a developer, I want the codebase to be free of unused or junk code, so that the system remains clean and maintainable.

#### Acceptance Criteria

1. WHEN code is no longer needed THEN the developer SHALL delete the unused code (no commented-out blocks, no dead methods)
2. WHEN refactoring THEN the developer SHALL remove deprecated methods and classes using @Deprecated annotation followed by deletion in next release
3. WHEN reviewing code THEN the reviewer SHALL identify and request removal of: unreachable code, unused imports, unused variables, unused dependencies
4. WHEN dependencies are unused THEN the developer SHALL remove them from pom.xml (Maven) or package.json (npm) using dependency analysis tools
5. WHEN a PR is submitted THEN the CI pipeline SHALL run static analysis (SpotBugs, ESLint) and fail on unused code warnings

### Requirement 45: Backup and Disaster Recovery

**User Story:** As a system administrator, I want automated backup and disaster recovery procedures, so that data can be recovered in case of catastrophic failures.

#### Acceptance Criteria

1. WHEN the system operates THEN the Business Network System SHALL perform automated daily backups of: PostgreSQL (pg_dump), Neo4j (neo4j-admin dump), Elasticsearch (snapshot API)
2. WHEN backups are created THEN the Business Network System SHALL store them in a separate region with 30-day retention and 90-day archive
3. WHEN disaster recovery is tested THEN the Business Network System SHALL restore from backup to a clean environment within 4 hours (RTO)
4. WHEN data loss occurs THEN the Business Network System SHALL recover to within 24 hours of the failure (RPO)
5. WHEN backups complete THEN the Business Network System SHALL verify integrity via restore test to staging environment (weekly automated test)

### Requirement 46: Data Retention Policy

**User Story:** As a system administrator, I want data retention policies enforced, so that the system complies with data governance requirements and manages storage costs.

#### Acceptance Criteria

1. WHEN QuickBooks Events age beyond 90 days THEN the Business Network System SHALL move them from hot storage (PostgreSQL) to cold storage (S3/GCS parquet files)
2. WHEN data ages beyond 1 year THEN the Business Network System SHALL archive to glacier storage with metadata index for retrieval
3. WHEN data ages beyond 7 years THEN the Business Network System SHALL permanently delete per compliance requirements (configurable per deployment)
4. WHEN retention policies execute THEN the Business Network System SHALL log: records_archived, records_deleted, storage_freed, execution_duration
5. WHEN querying archived data THEN the Business Network System SHALL provide an archive query API with 24-hour SLA for retrieval

### Requirement 47: Audit Logging

**User Story:** As a compliance officer, I want all entity changes logged with full audit trail, so that changes can be traced and accountability maintained.

#### Acceptance Criteria

1. WHEN a Business Entity is created, updated, or deleted THEN the Business Network System SHALL log: action, entity_id, user_id, timestamp, before_state (JSON), after_state (JSON)
2. WHEN a Relationship is created or deleted THEN the Business Network System SHALL log: action, relationship_id, source_id, target_id, user_id, timestamp
3. WHEN Entity Resolution merges entities THEN the Business Network System SHALL log: merge_action, source_entity_id, target_entity_id, confidence_score, system_initiated (boolean)
4. WHEN audit logs are stored THEN the Business Network System SHALL write to immutable audit_log table with: id, timestamp, action, entity_type, entity_id, user_id, changes (JSONB), correlation_id
5. WHEN audit logs are queried THEN the Business Network System SHALL provide API endpoint GET /api/v1/audit-log?entity_id=X&start_date=Y&end_date=Z with pagination

### Requirement 48: API Rate Limiting

**User Story:** As a system administrator, I want API rate limiting enforced, so that the system is protected from abuse and resources are fairly distributed.

#### Acceptance Criteria

1. WHEN API requests are made THEN the Business Network System SHALL enforce rate limits per authenticated user: 100 requests per minute for standard endpoints
2. WHEN rate limits are exceeded THEN the Business Network System SHALL return HTTP 429 with headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset (Unix timestamp)
3. WHEN expensive operations are requested (export, Graph RAG) THEN the Business Network System SHALL apply lower limits: 10 requests per minute
4. WHEN rate limiting is implemented THEN the Business Network System SHALL use Redis with sliding window algorithm for distributed rate limiting
5. WHEN monitoring rate limits THEN the Business Network System SHALL emit metrics: rate_limit_hits (counter by endpoint), rate_limit_remaining (gauge by user)

### Requirement 49: API Versioning

**User Story:** As a developer, I want API versioning enforced, so that breaking changes can be introduced without disrupting existing clients.

#### Acceptance Criteria

1. WHEN APIs are exposed THEN the Business Network System SHALL use URL path versioning: /api/v1/*, /api/v2/*
2. WHEN breaking changes are introduced THEN the Business Network System SHALL increment major version and maintain previous version for 6 months deprecation period
3. WHEN APIs are deprecated THEN the Business Network System SHALL return header: Deprecation: true, Sunset: <date> on old version endpoints
4. WHEN API documentation is generated THEN the Business Network System SHALL generate separate OpenAPI specs per version: openapi-v1.yaml, openapi-v2.yaml
5. WHEN clients use deprecated APIs THEN the Business Network System SHALL log: deprecated_api_call, endpoint, client_id, timestamp for migration tracking

### Requirement 50: Timeout Configuration

**User Story:** As a system architect, I want configurable timeouts for all external calls, so that slow dependencies don't cascade into system-wide failures.

#### Acceptance Criteria

1. WHEN calling Entity Resolution Service THEN the calling service SHALL timeout after 30 seconds and return partial results or cached data
2. WHEN calling Neo4j for graph traversal THEN the calling service SHALL timeout after 10 seconds and suggest reducing traversal depth
3. WHEN calling Elasticsearch THEN the calling service SHALL timeout after 5 seconds and fall back to direct Neo4j query
4. WHEN calling Vector Database THEN the calling service SHALL timeout after 5 seconds and proceed with keyword search only
5. WHEN timeouts occur THEN the Business Network System SHALL log: timeout_event, target_service, duration_ms, correlation_id and emit timeout_count metric

### Requirement 51: Centralized Logging and Distributed Tracing

**User Story:** As a system operator, I want centralized logging with distributed tracing across all microservices, so that I can quickly diagnose issues without navigating through multiple terminal windows.

#### Acceptance Criteria

1. WHEN any service logs a message THEN the Business Network System SHALL include trace_id and span_id in the log entry for request correlation across services
2. WHEN logs are generated THEN the Business Network System SHALL publish all logs to Kafka topic 'application.logs' in JSON format with fields: timestamp, level, service_name, trace_id, span_id, message, context
3. WHEN Logstash consumes log events THEN the system SHALL parse JSON logs, enrich with metadata, and index them in Elasticsearch index 'logs-*' with daily rotation
4. WHEN viewing logs in Kibana THEN the system SHALL provide a unified dashboard showing: log stream, error rate by service, request traces, and service health indicators
5. WHEN tracing a request THEN the Business Network System SHALL propagate trace_id through HTTP headers (X-B3-TraceId) and Kafka message headers enabling end-to-end request tracking

### Requirement 52: Python Service Logging Integration

**User Story:** As a system operator, I want Python services to integrate with the centralized logging system, so that AI Service logs are visible alongside Java service logs.

#### Acceptance Criteria

1. WHEN the Python AI Service logs a message THEN the system SHALL format logs as JSON with fields: timestamp, level, service_name, trace_id, span_id, message, context
2. WHEN the AI Service receives HTTP requests THEN the system SHALL extract trace_id from X-B3-TraceId header and include it in all subsequent log entries
3. WHEN the AI Service logs to stdout/stderr THEN the system SHALL publish logs to Kafka topic 'application.logs' using a Python Kafka producer
4. WHEN errors occur in Python services THEN the system SHALL log stack traces with trace_id for correlation with upstream Java services
5. WHEN the AI Service starts THEN the system SHALL log startup events with service version, configuration, and health status to the centralized logging system

### Requirement 53: Kibana Dashboards and Visualization

**User Story:** As a system operator, I want pre-configured Kibana dashboards, so that I can immediately visualize system health, errors, and request flows without manual dashboard creation.

#### Acceptance Criteria

1. WHEN Kibana starts THEN the Business Network System SHALL automatically provision dashboards: Service Overview, Error Analysis, Request Tracing, Performance Metrics, Kafka Consumer Lag
2. WHEN viewing the Service Overview dashboard THEN Kibana SHALL display: log volume by service (last 15 minutes), error rate by service, active trace count, service status indicators
3. WHEN viewing the Error Analysis dashboard THEN Kibana SHALL display: error count by service, error types, stack traces, affected trace IDs, error trends (last 24 hours)
4. WHEN viewing the Request Tracing dashboard THEN Kibana SHALL display: trace timeline visualization, service call graph, span durations, failed requests with trace IDs
5. WHEN searching logs THEN Kibana SHALL support queries by: trace_id, service_name, log level, time range, message content with autocomplete and syntax highlighting

### Requirement 54: End-to-End Data Flow Validation

**User Story:** As a system operator, I want to validate the complete end-to-end data flow from UI through all backend services, so that I can ensure all integration points work correctly before production deployment.

#### Acceptance Criteria

1. WHEN the UI triggers data ingestion THEN the Business Network System SHALL accept QuickBooks events via POST /api/v1/ingest/events and store them in PostgreSQL with status=PENDING
2. WHEN events are stored in PostgreSQL THEN the Ingest Flow SHALL publish events to appropriate Kafka topics (quickbooks.events.invoice, quickbooks.events.payment, quickbooks.events.vendor, quickbooks.events.customer) within 5 seconds
3. WHEN Kafka topics receive events THEN the Resolve Flow SHALL consume events, call Entity Resolution Service, create nodes in Neo4j, and publish entity.updated events to Kafka within 30 seconds
4. WHEN Neo4j nodes are created THEN the Search Service SHALL consume entity.updated events from Kafka and index entities in Elasticsearch within 10 seconds
5. WHEN the complete flow finishes THEN the UI SHALL be able to search for newly created entities via GET /api/v1/search/entities and retrieve results from Elasticsearch with Neo4j relationship data

### Requirement 55: UI-Driven Data Ingestion

**User Story:** As a developer testing the system, I want the UI to provide a data ingestion interface, so that I can trigger the complete data flow pipeline through user interactions.

#### Acceptance Criteria

1. WHEN the UI loads THEN the Business Network System SHALL provide a "Data Ingestion" page accessible from the navigation menu
2. WHEN a user accesses the Data Ingestion page THEN the system SHALL display a form to create QuickBooks events with fields: event_type (INVOICE|PAYMENT|VENDOR_ADD|CUSTOMER_ADD), source_business_name, source_business_address, target_business_name, target_business_address, amount
3. WHEN a user submits the ingestion form THEN the UI SHALL call POST /api/v1/ingest/events with the event data and display the response including event_id and status
4. WHEN ingestion succeeds THEN the UI SHALL display a success message with: "Event ingested successfully. Event ID: {id}. Track progress in the Flow Monitor."
5. WHEN the user navigates to the Flow Monitor page THEN the UI SHALL display real-time status of: events in PostgreSQL (PENDING/PROCESSING/PUBLISHED), Kafka topic message counts, Neo4j node counts, Elasticsearch document counts

### Requirement 56: End-to-End Flow Monitoring Dashboard

**User Story:** As a system operator, I want a real-time dashboard showing data flow progress through all pipeline stages, so that I can verify each integration point is working correctly.

#### Acceptance Criteria

1. WHEN the Flow Monitor dashboard loads THEN the Business Network System SHALL display current counts for: PostgreSQL events by status, Kafka topic message counts, Neo4j total nodes, Elasticsearch total documents, Vector DB total vectors
2. WHEN data flows through the pipeline THEN the dashboard SHALL refresh counts every 5 seconds automatically without page reload
3. WHEN displaying pipeline stages THEN the dashboard SHALL show visual indicators: PostgreSQL (green if events exist), Kafka (green if messages > 0), Neo4j (green if nodes > 0), Elasticsearch (green if documents > 0)
4. WHEN a user clicks on a pipeline stage THEN the dashboard SHALL display detailed information: recent events, error counts, processing times, sample data
      WHEN errors occur in any stage THEN the dashboard SHALL display error indicators (red) with error messages and affected event IDs for troubleshooting

### Requirement 57: Unified Application Startup & Validation

**User Story:** As a developer, I want a single, reliable command to start the entire stack on any OS, so that I can focus on development without debugging environment issues.

#### Acceptance Criteria

1. WHEN a developer runs `scripts/start.ps1` (Windows) or `scripts/start.sh` (Linux/Mac) THEN the system SHALL start all components: Infrastructure (Docker), Java Services, Python AI Service, and React Frontend
2. WHEN starting up THEN the script SHALL enforce dependency checks: logs directory exists, Docker is running, ports are free
3. WHEN all services are started THEN the script SHALL automatically execute a Deep Validation suite checking:
    - Infrastructure Health (Port 5432, 7474, 9200, 9092, etc.)
    - Backend Liveness (/actuator/health)
    - Functional Data Existence (Postgres row counts > 0, Neo4j node counts > 0)
4. WHEN functionality fails (e.g., UI reachable but API returning 500) THEN the startup script SHALL report a failure with specific logs
5. WHEN the startup completes successfully THEN the script SHALL output clear access URLs for: UI, Gateway, Neo4j Browser, Kafka UI
