# Charon - Snapchain API Extensions

Charon provides additional API and data infrastructure services for [Snapchain](https://github.com/farcasterxyz/snapchain) nodes, extending the base Farcaster Hub functionality with PostgreSQL storage, Redis caching, and GraphQL APIs.

## Overview

This repository contains Docker Compose override configurations that extend a base Snapchain node deployment (managed by the `snapchain.sh` installer script) with additional services:

- **PostgreSQL** (pgvector-enabled) - Persistent data storage for Farcaster messages and user data
- **Redis** - High-performance caching and queue management
- **Waypoint** - Farcaster Hub API client that syncs data to PostgreSQL
- **Backfill Services** - Background workers for historical data synchronization
- **PostGraphile** - GraphQL API over PostgreSQL schema with interactive GraphiQL interface

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Base Snapchain Setup                           │
│  - snapchain node (Farcaster Hub)               │
│  - statsd (metrics)                             │
│  - grafana (monitoring)                         │
└────────────────┬────────────────────────────────┘
                 │
                 │ docker-compose.override.yml
                 ▼
┌─────────────────────────────────────────────────┐
│  Charon Extensions (this repo)                  │
│  ┌────────────────────────────────────────────┐ │
│  │ postgres (data storage)                    │ │
│  │ redis (cache/queue)                        │ │
│  │ waypoint (API service)                     │ │
│  │ backfill workers (historical data)         │ │
│  └────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────┐ │
│  │ postgraphile (GraphQL API) :5000           │ │
│  │  - GraphiQL interface                      │ │
│  │  - Automatic schema from postgres          │ │
│  └────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## Prerequisites

1. **Base Snapchain Node**: A working Snapchain node installation

   - Managed by the `snapchain.sh` installer script
   - Running `docker-compose.yml` with snapchain service

2. **Docker & Docker Compose**: Version 2.0+ with override support

3. **System Resources**:
   - Minimum 8GB RAM (16GB+ recommended)
   - 100GB+ available disk space for database

## Quick Start

### 1. Clone this repository

```bash
git clone <repository-url> charon
cd charon
```

### 2. Access the GraphQL API

Once the services are running, you can access the GraphQL API:

**Interactive GraphiQL Interface:**
```
http://<host>:5000/graphiql
```

**GraphQL Endpoint:**
```
http://<host>:5000/graphql
```

**Example Query:**
```graphql
query GetRecentCasts {
  allCasts(
    first: 10
    orderBy: TIMESTAMP_DESC
    condition: { deletedAt: null }
  ) {
    nodes {
      hash
      fid
      text
      timestamp
    }
  }
}
```

**Example with cURL:**
```bash
curl -X POST http://<host>:5000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ allCasts(first: 10, orderBy: TIMESTAMP_DESC, condition: { deletedAt: null }) { nodes { fid text timestamp } } }"}'
```

**📖 For complete API documentation, see [GRAPHQL_SCHEMA.md](./GRAPHQL_SCHEMA.md)**

## Services

### PostgreSQL (pgvector/pgvector:pg17)

- **Port**: 5432
- **Database**: Stores Farcaster messages, user profiles, and relational data
- **Extensions**: pgvector for AI/ML embeddings support
- **Volume**: `postgres-data` for persistent storage

### Redis (redis:7)

- **Port**: 6379
- **Purpose**: Job queue for backfill workers, caching layer
- **Persistence**: AOF (Append-Only File) enabled
- **Volume**: `redis-data`

### Waypoint (officialunofficial/waypoint:latest)

- **Ports**: 8080 (REST API), 8000 (MCP service)
- **Function**: Subscribes to Snapchain hub events, stores to PostgreSQL
- **Features**:
  - Real-time message streaming from hub
  - MCP (Model Context Protocol) service for AI integrations
  - Configurable shard subscription

### Backfill Services

Two services work together for historical data population:

1. **backfill-queue**: Queues FIDs for backfill
2. **backfill-worker**: Processes queue and fetches historical messages

Both run under the `backfill` profile (opt-in).

### PostGraphile (graphile/postgraphile)

- **Port**: 5000
- **Function**: Auto-generates a complete GraphQL API from the PostgreSQL database schema
- **Features**:
  - GraphiQL interface for interactive query development
  - Automatic CRUD operations for all tables
  - Relay-style connections with cursor-based pagination
  - Real-time schema introspection
  - Support for filtering, ordering, and complex queries

**Access GraphiQL**: `http://<host>:5000/graphiql`

**GraphQL Endpoint**: `http://<host>:5000/graphql`

**Schema Documentation**: See [GRAPHQL_SCHEMA.md](./GRAPHQL_SCHEMA.md) for complete API documentation including:
- All available types and queries
- Example queries (recent casts, follows, user profiles)
- Performance optimization tips
- Pagination and filtering patterns

## Resources

- [Snapchain GitHub](https://github.com/farcasterxyz/snapchain)
- [Farcaster Protocol Documentation](https://docs.farcaster.xyz/)
- [Waypoint Documentation](https://github.com/officialunofficial/waypoint)
- [PostGraphile Documentation](https://www.graphile.org/postgraphile/)
- [GraphQL Schema Documentation](./GRAPHQL_SCHEMA.md) - Complete API reference with examples
