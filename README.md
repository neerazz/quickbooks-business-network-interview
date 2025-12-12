# Design a QuickBooks Business Network

## Introduction

In this system design problem, you are tasked with designing a system within Quickbooks that efficiently maps out a business network graph. You will identify relationships between businesses, focusing primarily on vendor-client connections. This system aims to help users visualize which business is selling to who and understand the network of their business interactions.

## Problem Scope

Design a system that enables QuickBooks users to efficiently navigate through their network of business relationships. This includes allowing them to view their vendors and clients, search for a specific business client-vendor relationship, and map the relationship between businesses on the network.

## Use Cases

Your system should support the following primary use cases:

1. **Business views their network:** A user can view a map of their business's network, highlighting vendors and clients.
2. **Business searches for a specific relationship:** A user can search for a specific business and understand their direct and indirect relationships.
3. **Business can grow the network:** A user can add a new business as a vendor/client of their business. The proposed new business may or may not already exist in the network, but with different descriptors (free-text category/name for example).
4. **Service maintains high availability:** The system should be highly available and responsive.

## Constraints and Assumptions

* Assume we're dealing with 1 million businesses.
* Each business can have up to 100 direct relationships (vendors or clients).
* The system should support 10 million relationship searches per month.
* Relationships are undirected and weighted by transaction volume.
* Traffic is not evenly distributed; some relationships are queried more frequently.

## Proposed Solution: QuickBooks Business Network

The [QuickBooks Business Network](https://docs.google.com/presentation/d/e/2PACX-1vTlzTAIxtlDJ8LA4sn8k-DWV25FL1UmI8UsXOfKUlSKrEtEMjObXJWwY28oPjYsAYYcGPXIiQ6H7ef1/pub?start=false&loop=false&delayms=3000&slide=id.p8) is designed to efficiently map and visualize business relationships within the QuickBooks ecosystem. The system leverages a graph database to represent businesses as nodes and their vendor-client relationships as edges, weighted by transaction volume.