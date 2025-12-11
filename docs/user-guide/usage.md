# User Guide

## Overview

This guide explains how to use the QuickBooks Business Network platform to manage business relationships, search for businesses, and leverage AI-powered insights.

## Getting Started

### Accessing the Application

1. Open your browser and navigate to `http://localhost:3000`
2. Log in with your credentials (for POC, use the user selector to switch contexts)

### Dashboard Overview

The main dashboard displays:

- **Network Graph**: Visual representation of business relationships
- **Search Bar**: Find businesses by name, address, or category
- **Quick Actions**: Common tasks like adding relationships
- **Metrics Panel**: SLA and performance statistics

## Core Features

### 1. Viewing the Network Graph

The network visualization shows businesses as nodes and relationships as edges.

**Interactions:**

- **Click a node**: View business details
- **Click an edge**: View relationship details
- **Scroll**: Zoom in/out
- **Drag**: Pan the view
- **Use depth slider**: Adjust how many hops to display (1-3)

### 2. Searching for Businesses

**Basic Search:**

1. Type a business name in the search bar
2. View suggestions as you type
3. Click a result to navigate to details

**Advanced Search:**

- Use filters for category, location, relationship type
- Combine multiple criteria for precise results

### 3. Managing Relationships

**Adding a Relationship:**

1. Navigate to Relationships → Add New
2. Select source business
3. Select target business
4. Choose relationship type (Vendor, Client)
5. Click Submit

**Removing a Relationship:**

1. Navigate to the relationship in the graph or list
2. Click the delete icon
3. Confirm the deletion

### 4. Using Graph RAG (AI Chat)

The Graph RAG feature allows natural language queries about the business network.

**Example Queries:**

- "Show me all vendors for Acme Corporation"
- "What businesses are connected to XYZ Inc within 2 hops?"
- "Find potential suppliers in the technology sector"

**How to Use:**

1. Click the Chat icon in the navigation
2. Type your question in natural language
3. Wait for the AI to process and respond
4. Click on referenced businesses to view details

### 5. Exporting Data

**JSON Export:**

1. Navigate to Network → Export
2. Select JSON format
3. Click Download

**CSV Export:**

1. Navigate to Network → Export
2. Select CSV format
3. Click Download

### 6. Entity Resolution Review

When the system identifies potential duplicate businesses:

1. Navigate to Entity Resolution → Pending Reviews
2. Review the match candidates
3. View confidence scores and matching evidence
4. Click Approve to merge, or Reject to keep separate
5. Provide feedback to improve future matches

## Common Workflows

### Finding Business Partners

1. Search for your business in the search bar
2. View the network graph centered on your business
3. Explore connected businesses by clicking nodes
4. Use filters to find specific relationship types (vendors, clients)

### Analyzing Supply Chain

1. Use Graph RAG: "Show the supply chain for [Business Name]"
2. Review the visualized chain in the graph
3. Export the data for further analysis

### Monitoring SLA Metrics

1. Navigate to Metrics → SLA Dashboard
2. View p50, p95, p99 response times
3. Check for any threshold breaches
4. Review historical trends

## Tips and Best Practices

1. **Use specific queries**: More specific natural language queries yield better results
2. **Review entity resolutions**: Regular review improves data quality
3. **Leverage exports**: Use CSV exports for spreadsheet analysis
4. **Bookmark frequently accessed businesses**: Use browser bookmarks for quick access

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `/` | Focus search bar |
| `Esc` | Close modal/dialog |
| `?` | Show help |
| `G N` | Go to Network |
| `G S` | Go to Search |
| `G C` | Go to Chat |

## Need Help?

- Check the [Troubleshooting Guide](../setup/troubleshooting.md)
- Review the [API Documentation](../api/openapi-v1.yaml)
- Contact support via the Help menu
