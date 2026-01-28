# Payment Status Check Service - Technical Requirements

> Complete technical specification for reimplementing the payment status check service in another durable execution framework (e.g., Restate).

---

## Table of Contents

1. [Overview](#1-overview)
2. [REST API Specification](#2-rest-api-specification)
3. [Workflow Specifications](#3-workflow-specifications)
4. [Activity Specifications](#4-activity-specifications)
5. [Data Models](#5-data-models)
6. [Configuration Parameters](#6-configuration-parameters)
7. [Business Logic Rules](#7-business-logic-rules)
8. [Error Handling Strategy](#8-error-handling-strategy)
9. [External Service Contracts](#9-external-service-contracts)
10. [Non-Functional Requirements](#10-non-functional-requirements)

---

## 1. Overview

### Purpose

Build a payment status check service that:
1. Accepts a list of payment IDs (up to 10,000)
2. Identifies which payment gateway handles each payment (via Elasticsearch)
3. Groups payments by gateway
4. Calls external services (IDB Facade + PGI Gateway) for each payment
5. Returns aggregated results with success/failure details

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PAYMENT STATUS CHECK FLOW                             │
└─────────────────────────────────────────────────────────────────────────────┘

INPUT: POST /payments/check-status
       { "paymentIds": ["pay_001", "pay_002", ..., "pay_100"] }
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │   PHASE 1: ES LOOKUP          │
                    │   ─────────────────────────   │
                    │   • Batch payments (size: 10) │
                    │   • Parallel lookups in batch │
                    │   • Get gateway for each      │
                    │   • Track lookup failures     │
                    └───────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
             ┌───────────┐   ┌───────────┐   ┌───────────┐
             │  Stripe   │   │   Adyen   │   │  PayPal   │
             │ 40 pays   │   │  35 pays  │   │  25 pays  │
             └───────────┘   └───────────┘   └───────────┘
                    │               │               │
                    ▼               ▼               ▼
                    ┌───────────────────────────────┐
                    │   PHASE 2: CHUNK & PROCESS    │
                    │   ─────────────────────────   │
                    │   • Chunk each gateway (5)    │
                    │   • Process gateways parallel │
                    │   • Chunks sequential/gateway │
                    └───────────────────────────────┘
                                    │
                    For each chunk: │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
          ┌─────────────────┐           ┌─────────────────────┐
          │ IDB Facade Call │           │ PGI Gateway Calls   │
          │ (batch: 5 IDs)  │──success──│ (sequential: 1 ID)  │
          │                 │           │                     │
          └─────────────────┘           └─────────────────────┘
                    │                               │
                    └───────────────┬───────────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │   PHASE 3: AGGREGATE          │
                    │   ─────────────────────────   │
                    │   • Collect all results       │
                    │   • Build success/fail maps   │
                    │   • Return final result       │
                    └───────────────────────────────┘
                                    │
                                    ▼
OUTPUT: GET /payments/check-status/{workflowId}
        {
          "status": "COMPLETED",
          "result": {
            "successful": { "stripe": [...], "adyen": [...] },
            "failed": { "stripe": [{ chunk, error, stage }] },
            "gatewayLookupFailed": ["pay_099"]
          }
        }
```

---

## 2. REST API Specification

### Endpoint 1: Start Payment Status Check

| Property | Value |
|----------|-------|
| **Path** | `/payments/check-status` |
| **Method** | `POST` |
| **Content-Type** | `application/json` |
| **Success Status** | `202 Accepted` |

#### Request Body

```json
{
  "paymentIds": ["payment_001", "payment_002", "payment_003"]
}
```

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `paymentIds` | `string[]` | Required, non-empty, max 10,000 items | List of payment IDs to check |

#### Success Response (202 Accepted)

```json
{
  "workflowId": "payment-check-550e8400-e29b-41d4-a716-446655440000",
  "status": "STARTED"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `workflowId` | `string` | Unique identifier for tracking (format: `payment-check-{UUID}`) |
| `status` | `string` | Always `"STARTED"` |

#### Error Response (400 Bad Request)

```json
{
  "error": "Validation failed",
  "details": {
    "paymentIds": "must not be empty"
  }
}
```

---

### Endpoint 2: Query Payment Status Check Progress

| Property | Value |
|----------|-------|
| **Path** | `/payments/check-status/{workflowId}` |
| **Method** | `GET` |
| **Success Status** | `200 OK` |
| **Not Found Status** | `404 Not Found` |

#### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `workflowId` | `string` | The workflow ID returned from start endpoint |

#### Success Response (200 OK) - Running

```json
{
  "workflowId": "payment-check-550e8400-e29b-41d4-a716-446655440000",
  "status": "RUNNING",
  "progress": {
    "totalPayments": 100,
    "gatewaysIdentified": 3,
    "chunksTotal": 20,
    "chunksCompleted": 12,
    "chunksFailed": 1,
    "currentPhase": "GATEWAY_PROCESSING"
  },
  "result": null
}
```

#### Success Response (200 OK) - Completed

```json
{
  "workflowId": "payment-check-550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "progress": {
    "totalPayments": 100,
    "gatewaysIdentified": 3,
    "chunksTotal": 20,
    "chunksCompleted": 20,
    "chunksFailed": 1,
    "currentPhase": "COMPLETED"
  },
  "result": {
    "successful": {
      "stripe": ["pay_001", "pay_002", "pay_003"],
      "adyen": ["pay_010", "pay_011"]
    },
    "failed": {
      "stripe": [
        {
          "chunkIndex": 3,
          "paymentIds": ["pay_015", "pay_016", "pay_017"],
          "error": "Connection timeout",
          "stage": "IDB"
        }
      ]
    },
    "gatewayLookupFailed": ["pay_099", "pay_100"]
  }
}
```

#### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `workflowId` | `string` | The workflow identifier |
| `status` | `enum` | `RUNNING`, `COMPLETED`, `FAILED`, `NOT_FOUND` |
| `progress` | `object` | Current progress (see below) |
| `result` | `object\|null` | Final result (null if still running) |

#### Progress Object

| Field | Type | Description |
|-------|------|-------------|
| `totalPayments` | `int` | Total number of payment IDs submitted |
| `gatewaysIdentified` | `int` | Number of unique gateways found |
| `chunksTotal` | `int` | Total chunks across all gateways |
| `chunksCompleted` | `int` | Chunks finished (success or failure) |
| `chunksFailed` | `int` | Chunks that failed |
| `currentPhase` | `string` | `ES_LOOKUP`, `GATEWAY_PROCESSING`, `AGGREGATING`, `COMPLETED` |

#### Not Found Response (404)

```json
{
  "workflowId": "payment-check-invalid-id",
  "status": "NOT_FOUND",
  "progress": null,
  "result": null
}
```

---

## 3. Workflow Specifications

### Workflow 1: PaymentStatusCheckWorkflow (Parent/Orchestrator)

#### Purpose
Orchestrates the entire payment status check process, coordinating ES lookups and gateway processing.

#### Method Signature

```
checkPaymentStatuses(input: PaymentStatusCheckInput) -> CheckStatusResult
```

#### Input

```
PaymentStatusCheckInput {
  paymentIds: string[]              // List of payment IDs to process
  config: WorkflowConfig {
    maxParallelEsQueries: int       // Default: 10
    maxPaymentsPerChunk: int        // Default: 5
    activityTimeoutSeconds: long    // Default: 30
    maxRetryAttempts: int           // Default: 3
  }
}
```

#### Output

```
CheckStatusResult {
  successful: Map<string, string[]>      // gateway -> list of successful payment IDs
  failed: Map<string, FailedChunk[]>     // gateway -> list of failed chunks
  gatewayLookupFailed: string[]          // payment IDs where ES lookup failed
}
```

#### State (for progress queries)

```
ProgressInfo {
  totalPayments: int
  gatewaysIdentified: int
  chunksTotal: int
  chunksCompleted: int
  chunksFailed: int
  currentPhase: string
}
```

#### Query Method

```
getProgress() -> ProgressInfo
```

#### Execution Steps

```
Step 1: Initialize
─────────────────
• Set currentPhase = "ES_LOOKUP"
• Set totalPayments = input.paymentIds.size

Step 2: ES Lookup Phase (batched parallel)
──────────────────────────────────────────
FOR each batch of paymentIds (batch size = maxParallelEsQueries):
    • Schedule ALL lookups in batch IN PARALLEL
    • Wait for ALL lookups in batch to complete
    • Collect results:
      - Success: Map payment -> gateway
      - Failure: Add to gatewayLookupFailed list
    • Continue to next batch

Step 3: Group by Gateway
────────────────────────
• Group successful lookups by gateway name
• Set gatewaysIdentified = number of unique gateways
• For each gateway: chunk payment IDs (chunk size = maxPaymentsPerChunk)
• Set chunksTotal = sum of all chunks across all gateways

Step 4: Gateway Processing Phase (parallel child workflows)
───────────────────────────────────────────────────────────
• Set currentPhase = "GATEWAY_PROCESSING"
• FOR each gateway IN PARALLEL:
    - Spawn child workflow: GatewayWorkflow
    - Input: gateway name, list of chunks
• Wait for ALL child workflows to complete
• Collect results from each child

Step 5: Aggregate Results
─────────────────────────
• Set currentPhase = "AGGREGATING"
• Build successful map from child results
• Build failed map from child results
• Set currentPhase = "COMPLETED"
• Return CheckStatusResult
```

---

### Workflow 2: GatewayWorkflow (Child Workflow)

#### Purpose
Process all chunks for a single gateway, calling IDB Facade and PGI Gateway for each chunk.

#### Method Signature

```
processGateway(gateway: string, chunks: string[][]) -> GatewayResult
```

#### Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `gateway` | `string` | Gateway name (e.g., "stripe", "adyen") |
| `chunks` | `string[][]` | List of chunks, each chunk is a list of payment IDs |

#### Output

```
GatewayResult {
  gateway: string
  successfulPaymentIds: string[]
  failedChunks: FailedChunk[]
}

FailedChunk {
  chunkIndex: int
  paymentIds: string[]
  error: string
  stage: string          // "IDB" or "PGI"
}
```

#### State (for progress queries)

```
ChunkProgress {
  totalChunks: int
  completedChunks: int
  currentChunkIndex: int
}
```

#### Query Method

```
getChunkProgress() -> ChunkProgress
```

#### Execution Steps

```
FOR each chunk (index = 0 to chunks.size - 1):   // SEQUENTIAL
    │
    ├─► Step A: Call IDB Facade (batch)
    │   ─────────────────────────────────
    │   • Call: idbFacade.notify(gateway, chunk.paymentIds)
    │   • If FAILS:
    │       - Record FailedChunk(index, paymentIds, error, stage="IDB")
    │       - Skip to next chunk (do NOT call PGI)
    │   • If SUCCESS:
    │       - Continue to Step B
    │
    └─► Step B: Call PGI Gateway (one-by-one)
        ─────────────────────────────────────
        FOR each paymentId in chunk:          // SEQUENTIAL
            • Call: pgiGateway.checkStatus(gateway, paymentId)
            • If FAILS:
                - Record failure (paymentId, error, stage="PGI")
            • If SUCCESS:
                - Add to successfulPaymentIds

    Update: completedChunks++

RETURN GatewayResult
```

---

## 4. Activity Specifications

### Activity 1: ElasticsearchActivities.getGatewayForPayment

#### Purpose
Look up which payment gateway handles a specific payment.

#### Signature

```
getGatewayForPayment(paymentId: string) -> GatewayInfo
```

#### Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `paymentId` | `string` | The payment ID to look up |

#### Output

```
GatewayInfo {
  paymentId: string
  gatewayName: string
}
```

#### External Call

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **URL** | `{elasticsearch.url}/{index}/_doc/{paymentId}` |
| **Example** | `GET http://localhost:9200/payments/_doc/pay_001` |

#### Response Parsing

```json
// Elasticsearch response
{
  "_index": "payments",
  "_id": "pay_001",
  "_source": {
    "gatewayName": "stripe",
    "amount": 5000,
    "currency": "USD"
  }
}

// Extract: _source.gatewayName -> "stripe"
```

#### Error Handling

| HTTP Status | Exception | Behavior |
|-------------|-----------|----------|
| `404` | `PaymentNotFoundException` | Payment not found in ES |
| `4xx` | `ElasticsearchException` | Client error |
| `5xx` | `ElasticsearchException` | Server error (retryable) |

#### Retry Configuration

| Setting | Value |
|---------|-------|
| Max Attempts | 3 |
| Initial Interval | 1 second |
| Backoff Coefficient | 2.0 |
| Max Interval | 10 seconds |
| Timeout | 30 seconds |

---

### Activity 2: PaymentGatewayActivities.callIdbFacade

#### Purpose
Notify IDB Facade about a batch of payments for a gateway.

#### Signature

```
callIdbFacade(gateway: string, paymentIds: string[]) -> void
```

#### Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `gateway` | `string` | Gateway name |
| `paymentIds` | `string[]` | Batch of payment IDs (max 5) |

#### Output

None (void) - success means no exception thrown.

#### External Call

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **URL** | `{idbFacade.url}/api/v1/payments/notify` |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "gatewayName": "stripe",
  "paymentIds": ["pay_001", "pay_002", "pay_003", "pay_004", "pay_005"]
}
```

#### Error Handling

| HTTP Status | Exception | Behavior |
|-------------|-----------|----------|
| `4xx` | `IdbFacadeException` | Client error |
| `5xx` | `IdbFacadeException` | Server error (retryable) |

#### Retry Configuration

| Setting | Value |
|---------|-------|
| Max Attempts | 3 |
| Initial Interval | 1 second |
| Backoff Coefficient | 2.0 |
| Max Interval | 10 seconds |
| Timeout | 30 seconds |

---

### Activity 3: PaymentGatewayActivities.callPgiGateway

#### Purpose
Check payment status via PGI Gateway for a single payment.

#### Signature

```
callPgiGateway(gateway: string, paymentId: string) -> void
```

#### Input

| Parameter | Type | Description |
|-----------|------|-------------|
| `gateway` | `string` | Gateway name |
| `paymentId` | `string` | Single payment ID |

#### Output

None (void) - success means no exception thrown.

#### External Call

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **URL** | `{pgiGateway.url}/api/v1/payments/{paymentId}/check-status` |
| **Header** | `X-Gateway-Name: {gateway}` |

#### Example

```
POST http://localhost:8080/api/v1/payments/pay_001/check-status
Headers:
  X-Gateway-Name: stripe
  Content-Type: application/json
```

#### Error Handling

| HTTP Status | Exception | Behavior |
|-------------|-----------|----------|
| `4xx` | `PgiGatewayException` | Client error |
| `5xx` | `PgiGatewayException` | Server error (retryable) |

#### Retry Configuration

| Setting | Value |
|---------|-------|
| Max Attempts | 3 |
| Initial Interval | 1 second |
| Backoff Coefficient | 2.0 |
| Max Interval | 10 seconds |
| Timeout | 30 seconds |

---

## 5. Data Models

### Request/Response DTOs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA MODEL HIERARCHY                               │
└─────────────────────────────────────────────────────────────────────────────┘

CheckStatusRequest                    CheckStatusStartResponse
├── paymentIds: string[]              ├── workflowId: string
                                      └── status: string = "STARTED"

CheckStatusQueryResponse
├── workflowId: string
├── status: WorkflowStatus
├── progress: ProgressInfo?
└── result: CheckStatusResult?

WorkflowStatus (enum)
├── RUNNING
├── COMPLETED
├── FAILED
└── NOT_FOUND
```

### Workflow Models

```
PaymentStatusCheckInput               WorkflowConfig
├── paymentIds: string[]              ├── maxParallelEsQueries: int = 10
└── config: WorkflowConfig            ├── maxPaymentsPerChunk: int = 5
                                      ├── activityTimeoutSeconds: long = 30
                                      └── maxRetryAttempts: int = 3

ProgressInfo                          CheckStatusResult
├── totalPayments: int                ├── successful: Map<string, string[]>
├── gatewaysIdentified: int           ├── failed: Map<string, FailedChunk[]>
├── chunksTotal: int                  └── gatewayLookupFailed: string[]
├── chunksCompleted: int
├── chunksFailed: int
└── currentPhase: string
```

### Gateway Models

```
GatewayInfo                           GatewayResult
├── paymentId: string                 ├── gateway: string
└── gatewayName: string               ├── successfulPaymentIds: string[]
                                      └── failedChunks: FailedChunk[]

FailedChunk                           ChunkProgress
├── chunkIndex: int                   ├── totalChunks: int
├── paymentIds: string[]              ├── completedChunks: int
├── error: string                     └── currentChunkIndex: int
└── stage: string  // "IDB" | "PGI"
```

---

## 6. Configuration Parameters

### Business Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `maxParallelEsQueries` | 10 | Max concurrent ES lookups per batch |
| `maxPaymentsPerChunk` | 5 | Max payment IDs per chunk to gateway |
| `activityTimeoutSeconds` | 30 | Timeout for each activity call |
| `maxRetryAttempts` | 3 | Max retry attempts for failed activities |

### External Service URLs

| Service | Config Key | Default |
|---------|------------|---------|
| Elasticsearch | `elasticsearch.url` | `http://localhost:9200` |
| Elasticsearch Index | `elasticsearch.index` | `payments` |
| IDB Facade | `idbFacade.url` | `http://localhost:8080` |
| PGI Gateway | `pgiGateway.url` | `http://localhost:8080` |

### Retry Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| Initial Interval | 1 second | First retry delay |
| Backoff Coefficient | 2.0 | Multiplier for exponential backoff |
| Max Interval | 10 seconds | Cap on retry delay |
| Max Attempts | 3 | Total attempts (1 initial + 2 retries) |

### Workflow ID Format

```
payment-check-{UUID}

Example: payment-check-550e8400-e29b-41d4-a716-446655440000
```

---

## 7. Business Logic Rules

### Batching & Chunking

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BATCHING & CHUNKING RULES                             │
└─────────────────────────────────────────────────────────────────────────────┘

ES LOOKUP PHASE:
────────────────
Input: 100 payment IDs
Batch size: 10 (maxParallelEsQueries)

Batch 1: [pay_001..pay_010] → 10 parallel ES calls → wait for all
Batch 2: [pay_011..pay_020] → 10 parallel ES calls → wait for all
...
Batch 10: [pay_091..pay_100] → 10 parallel ES calls → wait for all

Rule: Batches are SEQUENTIAL, calls within batch are PARALLEL


GATEWAY CHUNKING:
─────────────────
After ES lookup, grouped by gateway:
  Stripe: [pay_001, pay_005, pay_012, pay_015, pay_023, pay_027, pay_031]
  Adyen:  [pay_002, pay_003, pay_008]

Chunk size: 5 (maxPaymentsPerChunk)

Stripe chunks:
  Chunk 0: [pay_001, pay_005, pay_012, pay_015, pay_023]
  Chunk 1: [pay_027, pay_031]

Adyen chunks:
  Chunk 0: [pay_002, pay_003, pay_008]

Rule: Each gateway processed IN PARALLEL (separate child workflow)
Rule: Chunks within gateway processed SEQUENTIALLY
```

### Parallelism Matrix

| Operation | Parallelism Level | Constraint |
|-----------|-------------------|------------|
| ES lookup batches | Sequential | One batch at a time |
| ES lookups within batch | Parallel | Up to `maxParallelEsQueries` (10) |
| Gateway processing | Parallel | All gateways simultaneously |
| Chunk processing per gateway | Sequential | One chunk at a time |
| IDB Facade calls | N/A | One call per chunk (batch) |
| PGI Gateway calls per chunk | Sequential | One payment at a time |

### Processing Dependencies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DEPENDENCY CHAIN                                      │
└─────────────────────────────────────────────────────────────────────────────┘

ES Lookup must succeed BEFORE:
  └── Payment can be included in gateway processing

IDB Facade must succeed BEFORE:
  └── PGI Gateway calls can be made for that chunk

PGI failures do NOT block:
  └── Other PGI calls in same chunk
  └── Next chunk processing
```

### Failure Isolation

| Failure Type | Impact | Continues Processing? |
|--------------|--------|----------------------|
| Single ES lookup fails | That payment skipped | Yes, other payments continue |
| IDB Facade fails for chunk | Entire chunk fails | Yes, next chunk continues |
| PGI Gateway fails for payment | That payment fails | Yes, other payments in chunk continue |
| Child workflow fails | That gateway fails | Yes, other gateways continue |

---

## 8. Error Handling Strategy

### Exception Hierarchy

```
PaymentCheckException (base)
├── PaymentNotFoundException
│   └── When: ES returns 404 for payment ID
│   └── Data: paymentId
│
├── ElasticsearchException
│   └── When: ES returns error (4xx/5xx)
│   └── Data: paymentId, message
│
├── IdbFacadeException
│   └── When: IDB Facade returns error
│   └── Data: gateway, paymentIds[], message
│
└── PgiGatewayException
    └── When: PGI Gateway returns error
    └── Data: gateway, paymentId, message
```

### Error Recording

```
ES Lookup Failure:
─────────────────
• Add paymentId to: gatewayLookupFailed[]
• Do NOT process this payment further
• Continue with other payments

IDB Facade Failure:
──────────────────
• Create FailedChunk:
    chunkIndex: current chunk index
    paymentIds: all IDs in chunk
    error: exception message
    stage: "IDB"
• Skip PGI calls for this chunk
• Continue to next chunk

PGI Gateway Failure:
───────────────────
• Record individual payment failure
• Create FailedChunk (or add to existing):
    chunkIndex: current chunk index
    paymentIds: [failed payment ID]
    error: exception message
    stage: "PGI"
• Continue to next payment in chunk
```

### Retry Behavior

```
Activity fails
     │
     ▼
Attempt < maxAttempts?
     │
    YES ──► Wait (initialInterval * backoffCoefficient^attempt)
     │           │
     │           ▼
     │      Retry activity
     │           │
     │           └──► Success? ──► Continue workflow
     │                   │
     │                  NO
     │                   │
     └───────────────────┘
     │
    NO
     │
     ▼
Throw exception to workflow
     │
     ▼
Workflow catches and records failure
```

---

## 9. External Service Contracts

### Elasticsearch API

#### GET /{index}/_doc/{paymentId}

**Success Response (200):**
```json
{
  "_index": "payments",
  "_id": "pay_001",
  "_version": 1,
  "found": true,
  "_source": {
    "gatewayName": "stripe",
    "amount": 5000,
    "currency": "USD",
    "status": "pending",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

**Not Found Response (404):**
```json
{
  "_index": "payments",
  "_id": "pay_invalid",
  "found": false
}
```

---

### IDB Facade API

#### POST /api/v1/payments/notify

**Request:**
```json
{
  "gatewayName": "stripe",
  "paymentIds": ["pay_001", "pay_002", "pay_003"]
}
```

**Success Response (200):**
```json
{
  "status": "accepted",
  "processedCount": 3
}
```

**Error Response (500):**
```json
{
  "error": "Gateway unavailable",
  "code": "GATEWAY_ERROR"
}
```

---

### PGI Gateway API

#### POST /api/v1/payments/{paymentId}/check-status

**Headers:**
```
X-Gateway-Name: stripe
Content-Type: application/json
```

**Success Response (200):**
```json
{
  "paymentId": "pay_001",
  "status": "completed",
  "gatewayReference": "ch_1234567890"
}
```

**Error Response (500):**
```json
{
  "error": "Payment not found in gateway",
  "code": "PAYMENT_NOT_FOUND"
}
```

---

## 10. Non-Functional Requirements

### Durability Requirements

| Requirement | Description |
|-------------|-------------|
| **Crash Recovery** | Workflow must resume from last checkpoint after worker crash |
| **At-Least-Once** | Each activity must execute at least once (with retries) |
| **Progress Persistence** | Progress must be queryable even after failures |
| **Result Persistence** | Final result must be stored and queryable |

### Scalability Requirements

| Metric | Requirement |
|--------|-------------|
| Max Payment IDs per Request | 10,000 |
| Max Concurrent ES Queries | 10 per batch |
| Max Payments per Gateway Chunk | 5 |
| Expected Gateways | 1-20 |

### Performance Expectations

| Operation | Expected Duration |
|-----------|-------------------|
| Single ES Lookup | < 100ms |
| Single IDB Call | < 500ms |
| Single PGI Call | < 500ms |
| 100 Payments Total | ~30-60 seconds |
| 1000 Payments Total | ~5-10 minutes |

### Observability Requirements

| Requirement | Implementation |
|-------------|----------------|
| **Progress Tracking** | Query method returns real-time progress |
| **Phase Visibility** | Current phase exposed via progress |
| **Failure Details** | Failed chunks include stage and error message |
| **Logging** | Structured logs with workflow ID context |

### Idempotency

| Scenario | Behavior |
|----------|----------|
| Same workflow ID started twice | Second attempt fails (ID already exists) |
| Activity retried after timeout | Activity must be idempotent |
| Worker crash during activity | Activity re-executed on recovery |

---

## Appendix: Implementation Checklist

### Core Components

- [ ] REST Controller with validation
- [ ] Workflow orchestrator (parent)
- [ ] Gateway processor (child/sub-workflow)
- [ ] ES lookup activity
- [ ] IDB Facade activity
- [ ] PGI Gateway activity
- [ ] Progress query mechanism
- [ ] Result aggregation logic

### Configuration

- [ ] External service URLs configurable
- [ ] Batch/chunk sizes configurable
- [ ] Timeout values configurable
- [ ] Retry policy configurable

### Error Handling

- [ ] ES lookup failures tracked
- [ ] IDB failures mark entire chunk failed
- [ ] PGI failures mark individual payment failed
- [ ] All failures include stage information

### Testing

- [ ] Unit tests for business logic
- [ ] Integration tests with mocked services
- [ ] Failure scenario tests
- [ ] Recovery/replay tests