# NebGov Indexer

The NebGov indexer is a service that consumes on-chain governance events from the Stellar network and stores them in a PostgreSQL database. It exposes a REST API and WebSocket endpoint for historical governance analytics, enabling real-time tracking of proposals, votes, delegations, and other governance activities.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Event Flow](#event-flow)
- [Local Development Setup](#local-development-setup)
- [Docker Setup](#docker-setup)
- [Configuration](#configuration)
- [Event Types](#event-types)
- [Database Schema](#database-schema)
- [API Endpoints](#api-endpoints)
- [WebSocket Events](#websocket-events)
- [Monitoring](#monitoring)
- [Testing](#testing)

## Overview

The indexer performs the following functions:

- **Event Streaming**: Continuously polls the Stellar Soroban RPC for new governance events from the governor contract and related contracts (wrapper, treasury, liquidity)
- **Proposal State Tracking**: Maintains up-to-date state of all proposals including vote tallies, execution status, and cancellation status
- **Notification Triggers**: Broadcasts real-time events via WebSocket to connected clients
- **Historical Analytics**: Provides REST API endpoints for querying governance data with pagination and filtering

## Architecture

```mermaid
graph TB
    A[Stellar Network] -->|Soroban RPC| B[Indexer Service]
    B -->|Poll Events| A
    B -->|Store/Query| C[PostgreSQL Database]
    B -->|REST API| D[Backend/Frontend]
    B -->|WebSocket| E[Real-time Clients]
    
    subgraph "Indexer Components"
        B1[Event Processor]
        B2[REST API Server]
        B3[WebSocket Server]
        B4[Cache Layer]
    end
    
    B --> B1
    B --> B2
    B --> B3
    B --> B4
```

### Key Components

- **Event Processor** (`src/events.ts`): Fetches events from Stellar RPC, processes them based on event type, and updates the database
- **REST API Server** (`src/api.ts`): Express server providing HTTP endpoints for querying governance data
- **WebSocket Server** (`src/ws.ts`): Real-time event broadcasting to connected clients
- **Cache Layer** (`src/cache.ts`): In-memory caching for frequently accessed data
- **Database Layer** (`src/db.ts`): PostgreSQL connection and migration management

## Event Flow

```mermaid
sequenceDiagram
    participant S as Stellar Network
    participant I as Indexer
    participant D as PostgreSQL
    participant C as WebSocket Clients

    I->>S: Poll for events (every 5s)
    S-->>I: Return events batch
    loop For each event
        I->>I: Parse event type & data
        I->>D: Insert/update records
        I->>I: Invalidate cache
        I->>C: Broadcast via WebSocket
    end
    I->>D: Update last indexed ledger
```

## Local Development Setup

### Prerequisites

- Node.js 20+
- pnpm 9.0.0+
- PostgreSQL 16+

### Step 1: Install Dependencies

```bash
# From repository root
pnpm install
```

### Step 2: Set Up Database

```bash
# Start PostgreSQL (using Docker or local installation)
docker run --name nebgov-postgres \
  -e POSTGRES_USER=nebgov \
  -e POSTGRES_PASSWORD=nebgov \
  -e POSTGRES_DB=nebgov \
  -p 5432:5432 \
  -d postgres:16-alpine
```

### Step 3: Configure Environment Variables

Create a `.env` file in `packages/indexer/`:

```bash
cd packages/indexer
cp .env.example .env
```

Edit `.env` with your configuration:

```env
DATABASE_URL=postgres://nebgov:nebgov@localhost:5432/nebgov
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
GOVERNOR_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
CO_SPONSORSHIP_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
TIMELOCK_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
CONVICTION_VOTING_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
VOTE_ESCROW_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
VOTING_REWARDS_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
WRAPPER_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
TREASURY_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
TREASURY_SIMULATION_ACCOUNT=GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
LIQUIDITY_ADDRESS=CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
STELLAR_NETWORK_PASSPHRASE=Test SDF Network ; September 2015
POLL_INTERVAL_MS=5000
PORT=3001
HEALTH_LAG_THRESHOLD=100
```

### Step 4: Run Database Migrations

```bash
pnpm --filter @nebgov/indexer run migrate
```

To rollback migrations:

```bash
pnpm --filter @nebgov/indexer run migrate:down
```

### Step 5: Start the Indexer

```bash
pnpm --filter @nebgov/indexer run dev
```

The indexer will:
1. Connect to PostgreSQL and run migrations
2. Start the REST API on port 3001
3. Start the WebSocket server on `ws://localhost:3001/events`
4. Begin polling the Stellar network for events

### Step 6: Verify Health

```bash
curl http://localhost:3001/health
```

Expected response:

```json
{
  "status": "ok",
  "last_indexed_ledger": 12345,
  "current_ledger": 12350,
  "lag_ledgers": 5,
  "lag_seconds": 25,
  "total_proposals_indexed": 42,
  "total_votes_indexed": 1337,
  "total_delegates_indexed": 15,
  "uptime_seconds": 120,
  "timestamp": "2026-06-29T12:00:00.000Z"
}
```

## Configuration

### Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `DATABASE_URL` | Yes | PostgreSQL connection string | `postgres://nebgov:nebgov@localhost:5432/nebgov` |
| `STELLAR_RPC_URL` | Yes | Stellar Soroban RPC endpoint | `https://soroban-testnet.stellar.org` |
| `GOVERNOR_ADDRESS` | Yes | Governor contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `CO_SPONSORSHIP_ADDRESS` | Yes | Co-sponsorship contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `TIMELOCK_ADDRESS` | Yes | Timelock contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `CONVICTION_VOTING_ADDRESS` | Yes | Conviction voting contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `VOTE_ESCROW_ADDRESS` | Yes | Vote escrow contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `VOTING_REWARDS_ADDRESS` | Yes | Voting rewards contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `WRAPPER_ADDRESS` | No | Token wrapper contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `TOKEN_VOTES_ADDRESS` | No | Token votes contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `TREASURY_ADDRESS` | No | Treasury contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `TREASURY_SIMULATION_ACCOUNT` | No | Account for treasury simulations | `GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `LIQUIDITY_ADDRESS` | No | Liquidity contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `SIGNAL_ANCHOR_ADDRESS` | No | Signal anchor contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `TREASURY_STRATEGIES_ADDRESS` | No | Treasury strategies contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `OPTIMISTIC_GOVERNOR_ADDRESS` | No | Optimistic governor contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `PROPOSAL_BONDS_ADDRESS` | No | Proposal bonds contract address | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `STELLAR_NETWORK_PASSPHRASE` | Yes | Stellar network passphrase | `Test SDF Network ; September 2015` |
| `POLL_INTERVAL_MS` | No | Event polling interval in milliseconds | `5000` |
| `PORT` | No | HTTP server port | `3001` |
| `HEALTH_LAG_THRESHOLD` | No | Maximum acceptable ledger lag in blocks | `100` |