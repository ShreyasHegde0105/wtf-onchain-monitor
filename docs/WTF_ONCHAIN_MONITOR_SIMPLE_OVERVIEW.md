# WTF On-Chain Monitor — Simple Project Overview

## 1. What is this project?

**WTF On-Chain Monitor** is a service that watches the World Trade Future blockchain activity on **Sepolia**.

It takes data from the blockchain, cleans and stores it in **PostgreSQL**, checks that the database matches the blockchain, and provides an **API** for the application/dashboard.

### Simple flow

![Basic Architecture](../Arch%20Diagrams/Basic_Arch.drawio.png)

The main goal is:

> **Get blockchain data reliably, store it properly, and make sure the stored data is correct.**

---

## 2. Why do we need it?

Reading directly from the blockchain every time is slow and inconvenient for an application.

We need this service because:

- Payroll events need to be easy to query.
- ERC-20 transfers need to be tracked.
- RPC calls can fail or return duplicate data.
- Blockchain reorganizations can change recent blocks.
- Database data can become different from blockchain data.
- Admins need a clear view of payroll and transaction activity.

So the service acts as a reliable **bridge between the blockchain and the application**.

---

## 3. Main Goals

The service should:

1. Connect to Sepolia.
2. Read MonthlyPayroll events.
3. Read generic ERC-20 events.
4. Support historical backfill.
5. Keep watching new finalized blocks.
6. Store data in PostgreSQL.
7. Avoid duplicate records.
8. Recover from RPC failures.
9. Handle blockchain reorganizations.
10. Track transaction status.
11. Compare database data with blockchain data.
12. Provide API endpoints for the application.
13. Provide health, sync, and error information.

---

## 4. Current Blockchain Setup

### Network

```text
Sepolia
```

The network should be configurable.

### Payroll Contract

The currently selected payroll contract is:

```text
0x25a2aa23067B7cF5a991fC56cF76E8BFE03Cc6eC
```

The repository previously had another conflicting address, so the application must use a **validated configuration** instead of choosing an address automatically.

### ERC-20 Token

The WTF ERC-20 token address is currently **not established**.

Therefore:

```text
Use a normal ERC-20 token for development/testing.
```

Keep the token address configurable so the real token can be added later.

---

## 5. What We Index

### MonthlyPayroll

We need to track:

```text
EmployerAdded
EmployerRemoved

EmployeeAdded
EmployeeRemoved

PayrollFunded

SalaryClaimed
```

Important information includes:

- Employer
- Employee
- Salary
- Amount paid
- Fee
- Amount credited
- Block number
- Timestamp
- Transaction hash
- Log index

### ERC-20

Track standard:

```text
Transfer
Approval
```

The token address is configurable.

### Transactions

Also track:

```text
Transaction hash
Sender
Receiver
Gas
Status
Block
Confirmations
Error/revert information
```

---

## 6. Main Modules

### 6.1 Indexer

The **Indexer** reads blockchain data and puts it into the database.

It handles:

- Historical backfill
- Live syncing
- Event decoding
- Data normalization
- Checkpoints
- Retries
- Duplicate events
- Reorganizations

Simple idea:

```text
Blockchain
    ↓
Read events
    ↓
Decode
    ↓
Clean/normalize
    ↓
Save to DB
```

---

### 6.2 PostgreSQL / Persistence

PostgreSQL stores the indexed data.

Main tables are roughly:

```text
sync_checkpoints
chain_events
transactions
employers
employees
payroll_fundings
salary_claims
token_transfers
reconciliation_exceptions
```

We keep the original blockchain event information so that data can be rebuilt later if needed.

---

### 6.3 Reconciliation

Reconciliation is the **safety check**.

It compares:

```text
Blockchain = Source of Truth
        ↓
      Compare
        ↑
PostgreSQL = Indexed Copy
```

It helps find:

- Missing events
- Duplicate events
- Balance mismatches
- Payroll mismatches
- Token movement problems

Important:

> Reconciliation checks the indexer. It should not become a second indexer.

Also, comparisons should consider block height. A database that is simply behind the blockchain is not necessarily wrong.

---

### 6.4 API

The API gives the application easy access to the indexed data.

Main endpoints include:

```text
GET  /health
GET  /ready
GET  /v1/sync/status

POST /v1/sync/backfill

GET  /v1/transactions/{hash}
GET  /v1/employers/{address}
GET  /v1/employees/{address}

GET  /v1/payroll/fundings
GET  /v1/payroll/claims

GET  /v1/tokens/{address}/transfers

GET  /v1/reconciliation/exceptions
```

The API should support filtering, pagination, validation, and stable error responses.

---

### 6.5 Configuration

All important settings should come from one configuration system.

Examples:

```text
CHAIN_ID
RPC_URL
PAYROLL_CONTRACT_ADDRESS
TOKEN_ADDRESS
START_BLOCK
CONFIRMATION_DEPTH
DATABASE_URL
POLLING_INTERVAL
ENVIRONMENT
```

If required configuration is missing or invalid, the service should fail clearly.

---

## 7. Reliability

### No duplicates

The same blockchain event can sometimes be received more than once.

We identify an event using:

```text
chain_id
+ contract_address
+ transaction_hash
+ log_index
```

The database must prevent duplicates.

### Checkpoints

The indexer remembers where it stopped.

```text
Process blocks
      ↓
Save data
      ↓
Save checkpoint
```

If the service crashes:

```text
Restart
  ↓
Read checkpoint
  ↓
Continue
```

### RPC failures

If RPC fails:

```text
RPC failure
    ↓
Retry
    ↓
Backoff
    ↓
Retry again
```

The service must **not silently skip blocks**.

### Reorganizations

A blockchain reorg can replace recent blocks.

The service should:

```text
Detect reorg
    ↓
Identify affected events
    ↓
Mark old events removed/superseded
    ↓
Process canonical chain
    ↓
Fix affected projections
```

### Finality

The service should distinguish:

```text
Observed
```

from:

```text
Finalized / Safe
```

This prevents the dashboard from treating very recent blockchain activity as permanently final.

---

## 8. Data Provenance

Every indexed event must be traceable back to the blockchain.

The main identity is:

```text
chain_id
contract_address
transaction_hash
log_index
```

This lets us answer:

> Which blockchain event created this database record?

Raw event data should be retained so projections can be rebuilt if the application logic changes.

---

## 9. Security

The monitoring service is mainly a **read-only system**.

It must:

- Never accept private keys.
- Never accept signing authority.
- Validate addresses and chain IDs.
- Validate block ranges and numbers.
- Use parameterized SQL.
- Use least-privilege database access.
- Treat RPC data as untrusted input.
- Protect admin/operator endpoints.
- Never print secrets in logs.
- Redact credentials from connection strings.
- Avoid exposing secrets through health endpoints.

---

## 10. Observability

We should always be able to see what the indexer is doing.

Important information:

```text
Current block
Finalized block
Indexer lag
Last successful block
Last RPC error
Stream status
Reconciliation status
Open reconciliation issues
```

Use:

- Structured logs
- Metrics
- Health endpoint
- Readiness endpoint
- Sync-status endpoint

---

## 11. Testing

### Unit tests

Test things like:

```text
ABI decoding
Address validation
Number conversion
Pagination
Configuration validation
Reconciliation rules
```

### Integration tests

Test:

```text
Blockchain → Indexer → PostgreSQL
ERC-20 events
Transaction processing
RPC failures
```

### Important reliability tests

Test:

```text
Same range processed twice
        ↓
No duplicates
```

```text
RPC goes down
        ↓
Retry + recover
        ↓
No skipped blocks
```

```text
Blockchain reorg
        ↓
Old data corrected
```

### API tests

Test authentication, validation, filters, pagination, and errors.

---

## 12. Project Milestones

### M1 — Design & Configuration

- Architecture
- Configuration
- Confirmed addresses
- Data model

### M2 — Historical Indexer

- RPC adapter
- ABI decoder
- Database migrations
- Checkpointing
- Backfill

**Done when:** backfill works and can be safely repeated.

### M3 — Live Monitoring

- Finalized-block polling
- Retries
- Transaction status
- Health/sync endpoints

### M4 — Token & Reconciliation

- ERC-20 indexing
- Balance checks
- Mismatch detection

### M5 — Product Integration

- API client
- Dashboard data contract
- Employer/employee/transaction queries

### M6 — Hardening & Handoff

- Tests
- Docker
- Documentation
- Runbook
- Monitoring setup

---

## 13. Important Design Decisions

### Blockchain is the source of truth

```text
Blockchain
    ↓
Authoritative state

PostgreSQL
    ↓
Indexed/queryable copy
```

PostgreSQL makes the data fast and easy to query, while reconciliation checks that it has not drifted from the chain.

### Raw events are kept

```text
Blockchain
    ↓
Raw Events
    ↓
Projections
```

This allows projections to be rebuilt later.

### Configuration is centralized

One validated configuration should be shared by the Indexer, API, Persistence, and Reconciliation modules.

---

## 14. Non-Goals for Version 1

We are **not** building:

- A new payroll smart contract
- Wallet authentication
- Fiat on-ramp
- Exchange
- Custody system
- Investment/price decisions
- Compliance decisions

The goal is the **monitoring and data layer**.

---

## 15. Simple Project Definition

> **WTF On-Chain Monitor watches the Sepolia blockchain, collects payroll and token activity, stores it in PostgreSQL, checks that the database matches the blockchain, and provides the data through an API.**

### In one picture

```text
        BLOCKCHAIN
             ↓
       Read blockchain
             ↓
          INDEXER
             ↓
        PostgreSQL
         ↙       ↘
Reconciliation    API
      ↓             ↓
 Check correctness  App / Dashboard
```

---

## Source

Based on the **World Trade Future — On-Chain Payroll & WTF Token Monitoring Service** implementation specification.
