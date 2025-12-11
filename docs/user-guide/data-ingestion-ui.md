# Data Ingestion UI Guide

This guide explains how to use the Data Ingestion page to create test data and monitor the pipeline.

## Accessing the Data Ingestion Page

Navigate to: `http://localhost:3000/data-ingestion`

## Page Overview

### Metrics Dashboard

At the top of the page, you'll see real-time metrics:
- **Pending Events**: Events in PostgreSQL waiting to be processed
- **Published Events**: Events successfully published to Kafka
- **Outbox Pending**: Outbox entries waiting for publishing
- **Outbox Published**: Successfully published outbox entries

### Batch Data Generation

The recommended way to populate test data. Configure:

| Setting | Default | Description |
|---------|---------|-------------|
| Number of Businesses | 1000 | Total vendors (70%) and customers (30%) |
| Number of Transactions | 500 | Invoices (60%) and payments (40%) |
| Batch Size | 50 | Events per API call |

**To generate data:**
1. Adjust settings as needed
2. Click "🚀 Start Batch Generation"
3. Watch the progress bar
4. Click "View Flow Monitor" to track processing

### Single Event Ingestion

For testing individual events:

1. Expand "📝 Single Event Ingestion"
2. Fill in the form:
   - **Event Type**: INVOICE, PAYMENT, VENDOR_ADD, or CUSTOMER_ADD
   - **Source Business**: Name and optional category
   - **Target Business**: For transactions (INVOICE, PAYMENT)
   - **Amount**: Transaction amount (optional)
3. Click "Ingest Event"
4. Success message shows the Event ID

## Event Types

| Type | Description | Creates |
|------|-------------|---------|
| VENDOR_ADD | Register a new vendor | Business node |
| CUSTOMER_ADD | Register a new customer | Business node |
| INVOICE | Invoice between businesses | Relationship edge |
| PAYMENT | Payment between businesses | Relationship edge |

## Best Practices

1. **Start with businesses first**: Create VENDOR_ADD and CUSTOMER_ADD events before transactions
2. **Use batch generation for testing**: Much faster than single events
3. **Monitor the Flow Monitor**: Track data as it moves through the pipeline
4. **Verify via Search**: Confirm entities are searchable after processing

## Troubleshooting

### Events stuck as "Pending"
- Check that all services are running
- Verify Kafka is accessible
- Check the ingest-flow logs

### Batch generation fails
- Reduce batch size
- Check network connectivity
- Review browser console for errors

### Entity not appearing in search
- Wait for full pipeline processing (up to 30 seconds)
- Check Flow Monitor for processing status
- Verify in Neo4j and Elasticsearch directly
