# Restate Deep Dive for Senior Developers

> A technical deep-dive into Restate's internals, focusing on the journal, execution model, and how durable execution actually works. Includes detailed comparisons with Temporal. All examples in Kotlin.

---

## Table of Contents

1. [The Big Picture: What Problem Does Restate Solve?](#1-the-big-picture)
2. [Architecture: Restate vs Temporal](#2-architecture-restate-vs-temporal)
3. [The Journal: Core Concept (vs Temporal's Event Sourcing)](#3-the-journal-core-concept)
4. [Execution Model: How Code Actually Runs](#4-execution-model)
5. [Activities vs Side Effects: Deep Comparison](#5-activities-vs-side-effects)
6. [Determinism: What Must Be Deterministic and Why](#6-determinism)
7. [State Management: Implicit vs Explicit](#7-state-management)
8. [Virtual Objects vs Temporal Workflows](#8-virtual-objects-vs-temporal-workflows)
9. [Signals vs Awakeables: Communication Patterns](#9-signals-vs-awakeables)
10. [Service-to-Service Communication](#10-service-to-service-communication)
11. [Failure Scenarios and Recovery](#11-failure-scenarios-and-recovery)
12. [Versioning and Deployments](#12-versioning-and-deployments)
13. [Practical Examples](#13-practical-examples)

---

## 1. The Big Picture

### The Problem

Consider this code:

```kotlin
suspend fun processOrder(orderId: String): OrderResult {
    val payment = chargePayment(orderId)           // Step 1: HTTP call
    updateInventory(orderId)                        // Step 2: DB call
    sendConfirmationEmail(orderId, payment.id)      // Step 3: Email API
    return OrderResult(success = true, paymentId = payment.id)
}
```

**What happens if the process crashes after Step 2 but before Step 3?**

- Payment was charged ✓
- Inventory was updated ✓
- Email was NOT sent ✗
- The function has no memory of what happened

If you retry, you might:
- Charge the payment again (double charge!)
- Update inventory again (incorrect stock!)

### Restate's Solution

Restate makes your code **durable** - it remembers what happened and can resume from where it left off:

```kotlin
@Service
class OrderService {
    @Handler
    suspend fun processOrder(ctx: Context, orderId: String): OrderResult {
        val payment = ctx.runBlock { chargePayment(orderId) }
        ctx.runBlock { updateInventory(orderId) }
        ctx.runBlock { sendConfirmationEmail(orderId, payment.id) }
        return OrderResult(success = true, paymentId = payment.id)
    }
}
```

If crash happens after Step 2:
- Restate re-invokes the handler
- Steps 1 and 2 are **replayed from the journal** (not re-executed)
- Only Step 3 actually executes

**No double charges. No incorrect inventory. Automatic recovery.**

---

## 2. Architecture: Restate vs Temporal

### High-Level Architecture Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TEMPORAL ARCHITECTURE                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────────────────┐         ┌──────────────┐
│              │         │     TEMPORAL SERVER      │         │              │
│    CLIENT    │────────▶│                          │◀────────│    WORKER    │
│  (API/App)   │         │  • History storage       │  POLLS  │              │
│              │         │  • Task queues           │         │  • Workflows │
│  Starts      │         │  • State management      │         │  • Activities│
│  workflows   │         │  • Scheduling            │         │              │
└──────────────┘         └──────────────────────────┘         └──────────────┘
                                     │
                                     ▼
                         ┌──────────────────────┐
                         │      DATABASE        │
                         │  (PostgreSQL/MySQL/  │
                         │   Cassandra)         │
                         └──────────────────────┘

• Workers POLL the server for tasks (pull model)
• Full event history stored for each workflow
• Separate workflow and activity execution
• Requires external database


┌─────────────────────────────────────────────────────────────────────────────┐
│                           RESTATE ARCHITECTURE                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────────────────┐         ┌──────────────┐
│              │         │      RESTATE SERVER      │         │              │
│    CLIENT    │────────▶│                          │────────▶│   SERVICE    │
│  (API/App)   │   HTTP  │  • Journal storage       │  PUSHES │   HANDLER    │
│              │         │  • State management      │   HTTP  │              │
│  Invokes     │         │  • Durable execution     │         │  • Handlers  │
│  handlers    │         │  • Virtual objects       │         │  • Objects   │
└──────────────┘         └──────────────────────────┘         └──────────────┘
                                     │
                                     ▼
                         ┌──────────────────────────┐
                         │   EMBEDDED STORAGE       │
                         │   (RocksDB built-in)     │
                         └──────────────────────────┘

• Server PUSHES to services via HTTP (push model)
• Only journaled operations stored (not full history)
• Services are just HTTP endpoints
• Embedded storage (no external DB required)
```

### Key Architectural Differences

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **Communication model** | Pull (workers poll for tasks) | Push (server invokes handlers via HTTP) |
| **Worker lifecycle** | Long-running, stateful workers | Stateless HTTP handlers |
| **Task distribution** | Via task queues | Via HTTP routing to service endpoints |
| **Storage** | External DB (Postgres, MySQL, Cassandra) | Embedded RocksDB (or external) |
| **Deployment** | Server + Workers + Database | Server + Services |
| **Scaling workers** | Add more worker instances polling same queue | Standard HTTP service scaling |
| **Protocol** | gRPC (proprietary task protocol) | HTTP/gRPC (standard protocols) |

### What This Means In Practice

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// TEMPORAL: You define a Worker that polls for tasks
// ═══════════════════════════════════════════════════════════════════════════

fun main() {
    val client = WorkflowClient.newInstance(
        WorkflowServiceStubs.newServiceStubs()
    )

    val factory = WorkerFactory.newInstance(client)
    val worker = factory.newWorker("payment-task-queue")  // Task queue name

    // Register workflow and activity implementations
    worker.registerWorkflowImplementationTypes(PaymentWorkflowImpl::class.java)
    worker.registerActivitiesImplementations(PaymentActivitiesImpl())

    // Start polling (blocking)
    factory.start()
}

// Worker continuously polls Temporal server:
// "Any tasks for payment-task-queue?" → Execute → Report result → Poll again


// ═══════════════════════════════════════════════════════════════════════════
// RESTATE: You define an HTTP endpoint, Restate calls you
// ═══════════════════════════════════════════════════════════════════════════

fun main() {
    RestateHttpEndpointBuilder
        .builder()
        .bind(PaymentService())      // Just a class with handlers
        .bind(ShoppingCart())        // Another service
        .buildAndListen(9080)        // Standard HTTP server
}

// Then register with Restate server:
// restate deployments register http://localhost:9080

// Restate calls your service when needed:
// Restate → HTTP POST /PaymentService/process → Your handler executes
```

### Why Push vs Pull Matters

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TEMPORAL (Pull/Polling)                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Timeline:
─────────────────────────────────────────────────────────────────────────────

Worker                          Temporal Server
  │                                   │
  │ ──── Poll (any tasks?) ─────────> │
  │ <─────────── No ──────────────────│
  │                                   │
  │ ──── Poll (any tasks?) ─────────> │  ← Latency: polling interval
  │ <─────────── No ──────────────────│
  │                                   │
  │                                   │ ← Client starts workflow
  │                                   │
  │ ──── Poll (any tasks?) ─────────> │
  │ <──── Yes, WorkflowTask ──────────│  ← Task delivered
  │                                   │
  │       [Execute]                   │
  │                                   │
  │ ──── Complete + Commands ────────>│
  │                                   │

Characteristics:
• Latency depends on polling interval
• Workers maintain persistent connections
• Task queues provide natural load balancing
• More infrastructure (workers must be managed)


┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESTATE (Push/HTTP)                                       │
└─────────────────────────────────────────────────────────────────────────────┘

Timeline:
─────────────────────────────────────────────────────────────────────────────

Service                         Restate Server                    Client
  │                                   │                              │
  │                                   │ <──── Invoke handler ────────│
  │                                   │                              │
  │ <──── HTTP POST /handler ─────────│  ← Immediate invocation
  │                                   │
  │       [Execute]                   │
  │                                   │
  │ ───── HTTP Response ─────────────>│
  │                                   │ ───── Response ─────────────>│

Characteristics:
• Immediate invocation (no polling delay)
• Standard HTTP - works with any infrastructure
• Service scaling via standard load balancers
• Simpler deployment (just HTTP services)
```

---

## 3. The Journal: Core Concept

### What Is The Journal?

The journal is a **per-invocation append-only log** that records the results of operations that interact with the outside world or need to be deterministic across replays.

Think of it as a "save game" for your function execution.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           JOURNAL STRUCTURE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Invocation ID: inv_abc123                                                  │
│  Handler: OrderService/processOrder                                         │
│  Input: { orderId: "order_456" }                                            │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Journal Entries (append-only)                                        │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ Entry 0: Input         → { orderId: "order_456" }                   │   │
│  │ Entry 1: Run "charge"  → { paymentId: "pay_789", amount: 99.99 }    │   │
│  │ Entry 2: Run "inventory" → { success: true }                         │   │
│  │ Entry 3: Run "email"   → { messageId: "msg_012" }                   │   │
│  │ Entry 4: Output        → { success: true, paymentId: "pay_789" }    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What Gets Journaled?

| Operation | Journaled? | Why |
|-----------|------------|-----|
| `ctx.runBlock { }` | Yes | Side effects must not repeat on replay |
| `ctx.sleep()` | Yes | Timer state must be preserved |
| `ctx.get<T>()` / `ctx.set()` | Yes | State operations must be consistent |
| `ctx.awakeable<T>()` | Yes | External completion must be tracked |
| `ctx.call(service, handler, input)` | Yes | Service calls are durable |
| Regular Kotlin code | No | Doesn't need durability |
| `println()` / logging | No | No durability needed |
| Local variables | No | Reconstructed via replay |

### Journal vs Event Sourcing (Temporal) - THE KEY DIFFERENCE

This is the **most critical distinction** between Restate and Temporal. Understanding this unlocks everything else.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TEMPORAL (Event Sourcing)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Event History (records EVERYTHING):                                        │
│  ─────────────────────────────────────                                      │
│  Event 1:  WorkflowExecutionStarted                                         │
│  Event 2:  WorkflowTaskScheduled                                            │
│  Event 3:  WorkflowTaskStarted                                              │
│  Event 4:  WorkflowTaskCompleted                                            │
│  Event 5:  ActivityTaskScheduled { name: "charge" }                         │
│  Event 6:  ActivityTaskStarted                                              │
│  Event 7:  ActivityTaskCompleted { result: {...} }                          │
│  Event 8:  WorkflowTaskScheduled                                            │
│  Event 9:  WorkflowTaskStarted                                              │
│  Event 10: WorkflowTaskCompleted                                            │
│  ... (continues for every state transition)                                 │
│                                                                             │
│  On replay: Re-execute ALL workflow code from the beginning                 │
│             Match EVERY decision against history                            │
│             ALL code between activities must be deterministic               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESTATE (Journal)                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Journal (records ONLY ctx.* operations):                                   │
│  ─────────────────────────────────────────                                  │
│  Entry 0: Input → { orderId: "order_456" }                                  │
│  Entry 1: Run "charge" → { paymentId: "pay_789" }                           │
│  Entry 2: Run "inventory" → { success: true }                               │
│  Entry 3: Output → { success: true }                                        │
│                                                                             │
│  On replay: Re-execute handler code                                         │
│             When hitting ctx.runBlock(), return journaled result            │
│             Code BETWEEN ctx.* calls can be non-deterministic               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Insight: The Journal Is Your Checkpoint System

```kotlin
@Service
class PaymentService {
    @Handler
    suspend fun process(ctx: Context, input: PaymentRequest): PaymentResult {
        // Code here runs on every invocation/replay
        println("Starting...")  // Will print on every replay!

        // ═══════════════════════════════════════════════════════
        // CHECKPOINT 1: ctx.runBlock() - result is journaled
        // ═══════════════════════════════════════════════════════
        val result1 = ctx.runBlock {
            // This block ONLY runs if not in journal
            // On replay: returns journaled result immediately
            fetchExternalData()
        }

        // Code here runs on every invocation/replay
        val processed = result1.data.map { it * 2 }  // Runs every time

        // ═══════════════════════════════════════════════════════
        // CHECKPOINT 2: another ctx.runBlock()
        // ═══════════════════════════════════════════════════════
        val result2 = ctx.runBlock {
            saveToDatabase(processed)
        }

        return result2
    }
}
```

---

## 3. Execution Model

### How An Invocation Flows

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESTATE INVOCATION FLOW                                   │
└─────────────────────────────────────────────────────────────────────────────┘

     CLIENT                    RESTATE SERVER                    YOUR SERVICE
        │                            │                                │
        │  1. HTTP Request           │                                │
        │  POST /OrderService/process│                                │
        │ ──────────────────────────>│                                │
        │                            │                                │
        │                            │  2. Create invocation          │
        │                            │     Generate invocation ID     │
        │                            │     Initialize empty journal   │
        │                            │                                │
        │                            │  3. HTTP call to service       │
        │                            │ ──────────────────────────────>│
        │                            │                                │
        │                            │     Service executes handler   │
        │                            │     ctx.runBlock → journal     │
        │                            │     ctx.runBlock → journal     │
        │                            │                                │
        │                            │  4. Response (or suspension)   │
        │                            │ <──────────────────────────────│
        │                            │                                │
        │  5. Return result          │                                │
        │ <──────────────────────────│                                │
        │                            │                                │

KEY POINTS:
• Restate PUSHES to your service (HTTP call), not pull/poll like Temporal
• Journal is stored in Restate server, not your service
• Your service is stateless - state lives in Restate
```

### Suspension and Resumption

When your code hits `ctx.sleep()` or waits for an awakeable, the execution **suspends**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SUSPENSION FLOW                                           │
└─────────────────────────────────────────────────────────────────────────────┘

@Handler
suspend fun process(ctx: Context, input: Input): Result {
    val data = ctx.runBlock { fetchData() }

    // ════════════════════════════════════════════════════════════════════════
    // SUSPENSION POINT: ctx.sleep() triggers suspension
    // ════════════════════════════════════════════════════════════════════════
    ctx.sleep(Duration.ofMinutes(1))  // Sleep 1 minute

    ctx.runBlock { process(data) }
    return Result.success()
}

WHAT HAPPENS:

1. Handler executes until ctx.sleep()
2. SDK tells Restate: "suspend me, wake in 60s"
3. Restate journals the sleep entry
4. HTTP response sent back to Restate (handler "returns")
5. Your service's handler function EXITS (memory freed!)
6. ... 60 seconds pass ...
7. Restate re-invokes your handler (new HTTP request)
8. SDK replays from journal:
   - ctx.runBlock { fetchData() } → returns journaled result (no actual fetch)
   - ctx.sleep() → sees timer fired, continues immediately
9. Execution continues with ctx.runBlock { process(data) }

IMPORTANT: Local variables (like `data`) are RECONSTRUCTED by replaying
           the journal entries that produced them.
```

### The Replay Mechanism In Detail

```kotlin
// Original execution (first time)
@Handler
suspend fun process(ctx: Context, orderId: String): Result {
    println("A")                                          // Prints "A"

    val x = ctx.runBlock { api.call() }                   // Actually calls API
                                                          // Journals: { result: 42 }
    println("B: $x")                                      // Prints "B: 42"

    ctx.sleep(Duration.ofSeconds(10))                     // Suspends here
                                                          // Journals: { sleep: 10s }
    // --- EXECUTION STOPS HERE, handler exits ---

    println("C")                                          // NOT reached yet
    val y = ctx.runBlock { api.other() }
    return Result(y)
}

// After 10 seconds, REPLAY execution
@Handler
suspend fun process(ctx: Context, orderId: String): Result {
    println("A")                                          // Prints "A" again!

    val x = ctx.runBlock { api.call() }                   // Returns 42 from journal
                                                          // api.call() NOT invoked
    println("B: $x")                                      // Prints "B: 42" again!

    ctx.sleep(Duration.ofSeconds(10))                     // Journal says timer fired
                                                          // Continues immediately

    // --- REPLAY COMPLETE, now executing new code ---

    println("C")                                          // Prints "C" (first time!)
    val y = ctx.runBlock { api.other() }                  // Actually calls API
    return Result(y)
}
```

### Streaming Journal Protocol

The journal isn't sent all at once. Restate uses a **streaming protocol**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STREAMING JOURNAL PROTOCOL                                │
└─────────────────────────────────────────────────────────────────────────────┘

Restate Server                                      Your Service Handler
      │                                                      │
      │  HTTP Request with streaming body                    │
      │  Header: X-Restate-Invocation-Id: inv_123            │
      │ ─────────────────────────────────────────────────────>│
      │                                                      │
      │  Stream: [InputEntry: { orderId: "456" }]            │
      │ ─────────────────────────────────────────────────────>│
      │                                                      │
      │                      SDK parses, handler starts      │
      │                      handler hits ctx.runBlock()     │
      │                                                      │
      │  Stream: [RunEntry: { result: 42 }]     ← if replay  │
      │ ─────────────────────────────────────────────────────>│
      │                                                      │
      │       OR                                              │
      │                                                      │
      │  SDK executes block, sends result      ← if new      │
      │ <─────────────────────────────────────────────────────│
      │  Stream: [RunEntry: { result: 42 }]                  │
      │                                                      │

The bidirectional stream allows:
• Restate to send journal entries as needed
• SDK to send new journal entries as they're created
• Efficient incremental replay without loading full journal upfront
```

---

## 5. Activities vs Side Effects: Deep Comparison

This is where Temporal and Restate differ significantly in how you structure code.

### Temporal: Explicit Activity Interface

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// TEMPORAL: Activities are a separate concept from Workflows
// ═══════════════════════════════════════════════════════════════════════════

// Step 1: Define an interface
@ActivityInterface
interface PaymentActivities {
    @ActivityMethod
    fun chargePayment(paymentId: String, amount: Double): PaymentResult

    @ActivityMethod
    fun refundPayment(paymentId: String): RefundResult

    @ActivityMethod
    fun sendReceipt(email: String, paymentId: String)
}

// Step 2: Implement the interface
class PaymentActivitiesImpl(
    private val paymentGateway: PaymentGateway,
    private val emailService: EmailService
) : PaymentActivities {

    override fun chargePayment(paymentId: String, amount: Double): PaymentResult {
        // Real I/O happens here
        return paymentGateway.charge(paymentId, amount)
    }

    override fun refundPayment(paymentId: String): RefundResult {
        return paymentGateway.refund(paymentId)
    }

    override fun sendReceipt(email: String, paymentId: String) {
        emailService.send(email, "Payment $paymentId confirmed")
    }
}

// Step 3: Register with Worker
val worker = factory.newWorker("payment-queue")
worker.registerActivitiesImplementations(
    PaymentActivitiesImpl(paymentGateway, emailService)
)

// Step 4: Use in Workflow via stub
@WorkflowImpl(workers = ["payment-queue"])
class PaymentWorkflowImpl : PaymentWorkflow {

    // Create activity stub with options
    private val activities = Workflow.newActivityStub(
        PaymentActivities::class.java,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofSeconds(30))
            .setRetryOptions(
                RetryOptions.newBuilder()
                    .setInitialInterval(Duration.ofSeconds(1))
                    .setMaximumAttempts(5)
                    .setBackoffCoefficient(2.0)
                    .build()
            )
            .build()
    )

    override fun processPayment(paymentId: String, amount: Double): Result {
        // Activity calls look like regular method calls
        val payment = activities.chargePayment(paymentId, amount)
        activities.sendReceipt(payment.email, paymentId)
        return Result(payment.transactionId)
    }
}
```

### Restate: Inline Side Effects

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// RESTATE: Side effects are inline closures, no separate interface needed
// ═══════════════════════════════════════════════════════════════════════════

@Service
class PaymentService(
    private val paymentGateway: PaymentGateway,
    private val emailService: EmailService
) {

    @Handler
    suspend fun processPayment(ctx: Context, request: PaymentRequest): Result {
        // Side effect = ctx.runBlock { }
        // - Executes the closure
        // - Journals the result
        // - On replay, returns journaled result without executing

        val payment = ctx.runBlock {
            // Real I/O happens here
            paymentGateway.charge(request.paymentId, request.amount)
        }

        ctx.runBlock {
            emailService.send(payment.email, "Payment ${request.paymentId} confirmed")
        }

        return Result(payment.transactionId)
    }
}

// That's it. No interface, no registration, no stub.
```

### Side-by-Side: Same Logic, Different Patterns

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// THE SAME BUSINESS LOGIC IN BOTH FRAMEWORKS
// ═══════════════════════════════════════════════════════════════════════════

// Business requirement:
// 1. Reserve inventory
// 2. Charge payment
// 3. If payment fails, release inventory
// 4. Send confirmation email


// ─────────────────────────────────────────────────────────────────────────────
// TEMPORAL VERSION
// ─────────────────────────────────────────────────────────────────────────────

// Activities interface
@ActivityInterface
interface OrderActivities {
    fun reserveInventory(items: List<Item>): ReservationId
    fun releaseInventory(reservationId: ReservationId)
    fun chargePayment(userId: String, amount: Double): PaymentResult
    fun sendConfirmation(orderId: String, email: String)
}

// Workflow implementation
@WorkflowImpl
class OrderWorkflowImpl : OrderWorkflow {
    private val activities = Workflow.newActivityStub(
        OrderActivities::class.java,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofMinutes(1))
            .build()
    )

    override fun processOrder(order: Order): OrderResult {
        // Step 1
        val reservationId = activities.reserveInventory(order.items)

        // Step 2 with compensation
        val payment = try {
            activities.chargePayment(order.userId, order.total)
        } catch (e: ActivityFailure) {
            // Step 3: Compensate
            activities.releaseInventory(reservationId)
            throw e
        }

        // Step 4
        activities.sendConfirmation(order.id, order.email)

        return OrderResult(order.id, payment.transactionId)
    }
}


// ─────────────────────────────────────────────────────────────────────────────
// RESTATE VERSION
// ─────────────────────────────────────────────────────────────────────────────

@Service
class OrderService(
    private val inventoryApi: InventoryApi,
    private val paymentApi: PaymentApi,
    private val emailService: EmailService
) {

    @Handler
    suspend fun processOrder(ctx: Context, order: Order): OrderResult {
        // Step 1
        val reservationId = ctx.runBlock {
            inventoryApi.reserve(order.items)
        }

        // Step 2 with compensation
        val payment = try {
            ctx.runBlock {
                paymentApi.charge(order.userId, order.total)
            }
        } catch (e: Exception) {
            // Step 3: Compensate
            ctx.runBlock {
                inventoryApi.release(reservationId)
            }
            throw TerminalException("Payment failed", e)
        }

        // Step 4
        ctx.runBlock {
            emailService.send(order.email, "Order ${order.id} confirmed")
        }

        return OrderResult(order.id, payment.transactionId)
    }
}
```

### Key Differences Explained

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ACTIVITY vs SIDE EFFECT COMPARISON                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────┬───────────────────────────────────────────┐
│          TEMPORAL               │              RESTATE                       │
├─────────────────────────────────┼───────────────────────────────────────────┤
│                                 │                                           │
│ Separate interface required     │ No interface needed                       │
│                                 │                                           │
│ Activities registered on Worker │ Side effects are inline closures          │
│                                 │                                           │
│ Activity stub with options      │ ctx.runBlock { } with optional config     │
│                                 │                                           │
│ Can run on ANY worker          │ Runs in SAME process (or call service)    │
│ (distributed via task queue)    │                                           │
│                                 │                                           │
│ Each activity = separate task   │ Each ctx.runBlock = journal entry         │
│ in history                      │                                           │
│                                 │                                           │
│ Activity has its own timeout    │ runBlock can have timeout                 │
│ and retry policy               │ (or use global config)                    │
│                                 │                                           │
└─────────────────────────────────┴───────────────────────────────────────────┘

WHEN TO PREFER EACH:

Temporal Activities:
• When you need to distribute work across different workers
• When activities need different scaling than workflows
• When you want explicit interface contracts
• When activities are shared across multiple workflows

Restate Side Effects:
• Simpler code structure
• When side effects are specific to the handler
• When you want co-located execution
• When you want standard dependency injection
```

### Retry Configuration Comparison

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// TEMPORAL: Retry configured on activity stub
// ═══════════════════════════════════════════════════════════════════════════

private val activities = Workflow.newActivityStub(
    PaymentActivities::class.java,
    ActivityOptions.newBuilder()
        .setStartToCloseTimeout(Duration.ofSeconds(30))
        .setScheduleToCloseTimeout(Duration.ofMinutes(5))
        .setHeartbeatTimeout(Duration.ofSeconds(10))
        .setRetryOptions(
            RetryOptions.newBuilder()
                .setInitialInterval(Duration.ofSeconds(1))
                .setBackoffCoefficient(2.0)
                .setMaximumInterval(Duration.ofSeconds(60))
                .setMaximumAttempts(5)
                .setDoNotRetry(
                    IllegalArgumentException::class.java.name,
                    InvalidPaymentException::class.java.name
                )
                .build()
        )
        .build()
)


// ═══════════════════════════════════════════════════════════════════════════
// RESTATE: Retry is automatic (infinite by default), use TerminalException to stop
// ═══════════════════════════════════════════════════════════════════════════

// Default: infinite retry with exponential backoff
val result = ctx.runBlock {
    paymentApi.charge(amount)  // Retried forever until success
}

// To stop retrying, throw TerminalException
val result = ctx.runBlock {
    try {
        paymentApi.charge(amount)
    } catch (e: InvalidPaymentException) {
        // Don't retry this - it's a permanent failure
        throw TerminalException("Invalid payment", e)
    }
    // Other exceptions → automatic retry
}

// Or configure globally in Restate server settings for retry policy
```

---

## 6. Determinism

### What Must Be Deterministic?

**Only the SEQUENCE and PARAMETERS of ctx.* calls must be deterministic.**

```kotlin
// ✅ SAFE: Non-deterministic code OUTSIDE ctx.runBlock()
@Handler
suspend fun process(ctx: Context, input: Input): Result {
    val timestamp = System.currentTimeMillis()    // ✅ OK - not journaled
    val random = Random.nextDouble()              // ✅ OK - not journaled
    val uuid = UUID.randomUUID()                  // ✅ OK - not journaled

    println("Processing at $timestamp")           // Different each replay, but OK

    // ⚠️ The ctx.runBlock() call order must be deterministic
    val result = ctx.runBlock {
        // Inside here, code executes only once (or not at all on replay)
        api.call()
    }

    return Result(result)
}

// ❌ BROKEN: Non-deterministic ctx.* call sequence
@Handler
suspend fun process(ctx: Context, input: Input): Result {
    if (Random.nextDouble() > 0.5) {
        ctx.runBlock { doA() }  // Sometimes this is first
    } else {
        ctx.runBlock { doB() }  // Sometimes this is first
    }
    // On replay, journal has doA but code might try doB → MISMATCH!
}

// ✅ SAFE: Deterministic sequence, non-deterministic data inside
@Handler
suspend fun process(ctx: Context, input: Input): Result {
    // Move non-determinism INSIDE ctx.runBlock()
    val result = ctx.runBlock {
        if (Random.nextDouble() > 0.5) {
            doA()
        } else {
            doB()
        }
    }
    // Now the random choice is journaled, replay returns same result
}
```

### The Determinism Rule Visualized

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DETERMINISM REQUIREMENTS                                  │
└─────────────────────────────────────────────────────────────────────────────┘

@Handler
suspend fun process(ctx: Context, input: Input): Result {
  ┌──────────────────────────────────────────────────────────────────────────┐
  │ ZONE A: Regular code - CAN be non-deterministic                          │
  │                                                                          │
  │   val x = Random.nextDouble()  // ✅ Different each replay              │
  │   val t = System.currentTimeMillis()  // ✅ Different each replay       │
  │   println("hello")             // ✅ Runs each replay                   │
  │                                                                          │
  └──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ CHECKPOINT: ctx.runBlock { ... }                                         ║
  ║                                                                          ║
  ║   • The POSITION in sequence must be deterministic                       ║
  ║   • Block executes only on first run, skipped on replay                  ║
  ║   • Result is journaled and returned on replay                           ║
  ║                                                                          ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                                    │
                                    ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │ ZONE B: Regular code - CAN be non-deterministic                          │
  │                                                                          │
  │   // x came from journaled ctx.runBlock, so it's deterministic now       │
  │   val processed = x.map { ... }  // ✅ Same input → same output         │
  │                                                                          │
  └──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ CHECKPOINT: ctx.runBlock { ... }                                         ║
  ╚══════════════════════════════════════════════════════════════════════════╝
}
```

### Common Determinism Pitfalls

```kotlin
// ❌ PITFALL 1: Conditional ctx.* calls based on non-journaled data
@Handler
suspend fun process(ctx: Context, input: Input): Result {
    val config = fetchConfig()  // NOT journaled!
    if (config.enableFeatureX) {
        ctx.runBlock { doFeatureX() }
    }
    // If config changes between original and replay → mismatch!
}

// ✅ FIX: Journal the external state
@Handler
suspend fun process(ctx: Context, input: Input): Result {
    val config = ctx.runBlock { fetchConfig() }
    if (config.enableFeatureX) {
        ctx.runBlock { doFeatureX() }
    }
    // Now config is journaled, replay uses same value
}

// ❌ PITFALL 2: Non-deterministic iteration order
@Handler
suspend fun process(ctx: Context, items: Map<String, Item>): Result {
    for ((key, item) in items) {
        // Map iteration order might vary
        ctx.runBlock { processItem(key, item) }
    }
}

// ✅ BETTER: Explicit ordering
@Handler
suspend fun process(ctx: Context, items: Map<String, Item>): Result {
    val sortedKeys = items.keys.sorted()
    for (key in sortedKeys) {
        ctx.runBlock { processItem(key, items[key]!!) }
    }
}
```

---

## 7. State Management: Implicit vs Explicit

This is another fundamental difference. Temporal and Restate have completely different approaches to state.

### Temporal: Implicit State via Replay

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// TEMPORAL: State is stored in workflow fields, reconstructed via replay
// ═══════════════════════════════════════════════════════════════════════════

@WorkflowImpl
class OrderWorkflowImpl : OrderWorkflow {
    // State lives in workflow fields
    private var status = "PENDING"
    private var processedItems = 0
    private val failedItems = mutableListOf<String>()
    private var paymentId: String? = null

    override fun processOrder(items: List<String>): OrderResult {
        items.forEach { item ->
            try {
                activities.processItem(item)
                processedItems++
                status = "PROCESSING"
            } catch (e: ActivityFailure) {
                failedItems.add(item)
            }
        }

        if (failedItems.isEmpty()) {
            paymentId = activities.chargePayment()
            status = "COMPLETED"
        } else {
            status = "PARTIALLY_FAILED"
        }

        return OrderResult(status, processedItems, failedItems)
    }

    // Query methods return current state
    @QueryMethod
    override fun getStatus(): String = status

    @QueryMethod
    override fun getProgress(): Progress = Progress(processedItems, failedItems.size)
}

// HOW STATE WORKS IN TEMPORAL:
//
// 1. Workflow starts, fields initialized
// 2. processItem() succeeds → processedItems = 1, status = "PROCESSING"
//    Event recorded: ActivityTaskCompleted { result: ... }
// 3. Worker crashes
// 4. New worker picks up workflow
// 5. REPLAY: Workflow code runs from beginning
//    - processItem() call → SDK matches to history → returns cached result
//    - processedItems++ executes → processedItems = 1 (same as before)
//    - status = "PROCESSING" executes → same as before
// 6. State is reconstructed to exactly where it was
```

### Restate: Explicit State via API

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// RESTATE: State is explicitly stored via ctx.get/set
// ═══════════════════════════════════════════════════════════════════════════

@VirtualObject
class OrderProcessor {

    @Handler
    suspend fun processOrder(ctx: ObjectContext, items: List<String>): OrderResult {
        // Initialize state (or read existing)
        ctx.set(STATUS, "PENDING")
        ctx.set(PROCESSED_COUNT, 0)
        ctx.set(FAILED_ITEMS, emptyList<String>())

        items.forEach { item ->
            try {
                ctx.runBlock { processItem(item) }

                // Explicitly update state
                val count = ctx.get<Int>(PROCESSED_COUNT) ?: 0
                ctx.set(PROCESSED_COUNT, count + 1)
                ctx.set(STATUS, "PROCESSING")

            } catch (e: Exception) {
                val failed = ctx.get<List<String>>(FAILED_ITEMS) ?: emptyList()
                ctx.set(FAILED_ITEMS, failed + item)
            }
        }

        val failedItems = ctx.get<List<String>>(FAILED_ITEMS) ?: emptyList()
        if (failedItems.isEmpty()) {
            val paymentId = ctx.runBlock { chargePayment() }
            ctx.set(PAYMENT_ID, paymentId)
            ctx.set(STATUS, "COMPLETED")
        } else {
            ctx.set(STATUS, "PARTIALLY_FAILED")
        }

        return OrderResult(
            status = ctx.get(STATUS)!!,
            processedCount = ctx.get(PROCESSED_COUNT)!!,
            failedItems = failedItems
        )
    }

    // Shared handler for reading state (like Query)
    @Shared
    @Handler
    suspend fun getStatus(ctx: SharedObjectContext): String {
        return ctx.get(STATUS) ?: "UNKNOWN"
    }

    @Shared
    @Handler
    suspend fun getProgress(ctx: SharedObjectContext): Progress {
        return Progress(
            processedCount = ctx.get(PROCESSED_COUNT) ?: 0,
            failedCount = (ctx.get<List<String>>(FAILED_ITEMS) ?: emptyList()).size
        )
    }

    companion object {
        private val STATUS = StateKey.of<String>("status")
        private val PROCESSED_COUNT = StateKey.of<Int>("processedCount")
        private val FAILED_ITEMS = StateKey.of<List<String>>("failedItems")
        private val PAYMENT_ID = StateKey.of<String>("paymentId")
    }
}

// HOW STATE WORKS IN RESTATE:
//
// 1. Handler starts, ctx.set(STATUS, "PENDING") → stored in Restate
// 2. processItem() succeeds → ctx.set(PROCESSED_COUNT, 1) → stored
//    Journal entry: SetState { key: "processedCount", value: 1 }
// 3. Service crashes
// 4. Restate re-invokes handler
// 5. REPLAY: Handler runs, but ctx.get/set return journaled values
//    - ctx.set(STATUS, "PENDING") → journal says already done, skip
//    - ctx.runBlock { processItem() } → journal has result, return it
//    - ctx.set(PROCESSED_COUNT, 1) → journal says already done, skip
// 6. After replay, continues from where it left off
// 7. State is ALSO directly readable from storage (no replay needed for queries)
```

### State Comparison Table

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STATE MANAGEMENT COMPARISON                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────┬───────────────────────────────────────────┐
│          TEMPORAL               │              RESTATE                       │
├─────────────────────────────────┼───────────────────────────────────────────┤
│                                 │                                           │
│ STORAGE:                        │ STORAGE:                                  │
│ Event history (append-only)     │ Key-value store (mutable)                 │
│                                 │                                           │
│ STATE ACCESS:                   │ STATE ACCESS:                             │
│ Via class fields                │ Via ctx.get<T>(key) / ctx.set(key, val)   │
│                                 │                                           │
│ STATE RECONSTRUCTION:           │ STATE RECONSTRUCTION:                     │
│ Full replay of workflow code    │ Direct read from storage                  │
│                                 │                                           │
│ QUERY STATE:                    │ QUERY STATE:                              │
│ @QueryMethod (may need replay)  │ @Shared handler (direct read, no replay)  │
│                                 │                                           │
│ STATE PERSISTENCE:              │ STATE PERSISTENCE:                        │
│ Implicit (via history)          │ Explicit (ctx.set writes to storage)      │
│                                 │                                           │
│ CROSS-INVOCATION STATE:         │ CROSS-INVOCATION STATE:                   │
│ Signals + class fields          │ Virtual Object state persists             │
│                                 │                                           │
│ STATE SIZE LIMIT:               │ STATE SIZE LIMIT:                         │
│ History size (50k events)       │ Per-key storage limits (configurable)     │
│                                 │                                           │
└─────────────────────────────────┴───────────────────────────────────────────┘
```

### Why This Matters

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PRACTICAL IMPLICATIONS                                    │
└─────────────────────────────────────────────────────────────────────────────┘

TEMPORAL (Implicit State):
─────────────────────────
✅ Feels like normal programming (just use fields)
✅ No boilerplate for simple state
❌ Must understand replay semantics
❌ All code must be deterministic
❌ Queries may trigger replay (slow for large histories)
❌ History grows unbounded → need Continue-As-New

RESTATE (Explicit State):
─────────────────────────
✅ State reads are fast (direct from storage)
✅ Only ctx.* calls need to be deterministic
✅ State persists across handler invocations
✅ Clear separation: journal (execution) vs state (data)
❌ More verbose (explicit get/set)
❌ Need to think about state keys
❌ State schema changes need migration

EXAMPLE: Long-running order with 10,000 items

TEMPORAL:
- History: ~30,000 events (3 per item)
- Query getProgress(): Must replay 30k events
- Eventually hits limit → Continue-As-New required

RESTATE:
- Journal: ~10,000 entries (1 per item)
- Query getProgress(): Direct read from state key
- State stays small: just current values
```

---

## 8. Virtual Objects vs Temporal Workflows

### Virtual Objects = Temporal Workflow + Entity Pattern

Virtual Objects are **keyed stateful entities** that combine aspects of Temporal workflows with the entity/actor pattern. Think of them as:
- Have a unique key (like a primary key)
- Have persistent state (survives across invocations)
- Process one exclusive handler at a time per key (concurrency control)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VIRTUAL OBJECT CONCEPTUAL MODEL                           │
└─────────────────────────────────────────────────────────────────────────────┘

Virtual Object Definition: "ShoppingCart"

                    ┌─────────────────────────────────────┐
                    │        ShoppingCart Object          │
                    │           (Definition)              │
                    │                                     │
                    │  Handlers:                          │
                    │  ├─ addItem (exclusive)             │
                    │  ├─ removeItem (exclusive)          │
                    │  ├─ checkout (exclusive)            │
                    │  └─ getContents (shared)            │
                    └─────────────────────────────────────┘
                                     │
                                     │ instantiated per key
                                     ▼
    ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │ ShoppingCart    │  │ ShoppingCart    │  │ ShoppingCart    │
    │ key="user-123"  │  │ key="user-456"  │  │ key="user-789"  │
    │                 │  │                 │  │                 │
    │ State:          │  │ State:          │  │ State:          │
    │ ├─items: [...]  │  │ ├─items: [...]  │  │ ├─items: []     │
    │ └─total: 99.99  │  │ └─total: 45.00  │  │ └─total: 0      │
    │                 │  │                 │  │                 │
    │ Locked: NO      │  │ Locked: YES     │  │ Locked: NO      │
    │ (ready for      │  │ (running        │  │ (ready for      │
    │  requests)      │  │  checkout)      │  │  requests)      │
    └─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Exclusive vs Shared Handlers

```kotlin
@VirtualObject
class ShoppingCart {

    // ════════════════════════════════════════════════════════════════════
    // EXCLUSIVE HANDLER: Only one can run per key at a time
    // ════════════════════════════════════════════════════════════════════
    @Handler
    suspend fun addItem(ctx: ObjectContext, item: Item): CartSummary {
        // ctx.key() is the object key (e.g., "user-123")
        val items = ctx.get<List<Item>>(ITEMS) ?: emptyList()
        val newItems = items + item
        ctx.set(ITEMS, newItems)

        val total = newItems.sumOf { it.price }
        ctx.set(TOTAL, total)

        return CartSummary(itemCount = newItems.size, total = total)
    }

    @Handler
    suspend fun checkout(ctx: ObjectContext): CheckoutResult {
        val items = ctx.get<List<Item>>(ITEMS) ?: emptyList()
        val total = ctx.get<Double>(TOTAL) ?: 0.0

        // Process payment...
        val paymentResult = ctx.runBlock { paymentApi.charge(ctx.key(), total) }

        // Clear cart
        ctx.clear(ITEMS)
        ctx.clear(TOTAL)

        return CheckoutResult(paymentId = paymentResult.id, total = total)
    }

    // ════════════════════════════════════════════════════════════════════
    // SHARED HANDLER: Can run concurrently with other shared handlers
    // Cannot run while exclusive handler is running
    // ════════════════════════════════════════════════════════════════════
    @Shared
    @Handler
    suspend fun getContents(ctx: SharedObjectContext): CartContents {
        // Note: SharedObjectContext, not ObjectContext
        // Can only READ state, not write
        return CartContents(
            items = ctx.get<List<Item>>(ITEMS) ?: emptyList(),
            total = ctx.get<Double>(TOTAL) ?: 0.0
        )
    }

    companion object {
        private val ITEMS = StateKey.of<List<Item>>("items")
        private val TOTAL = StateKey.of<Double>("total")
    }
}
```

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VIRTUAL OBJECT CONCURRENCY                                │
└─────────────────────────────────────────────────────────────────────────────┘

Timeline for ShoppingCart key="user-123":

Time ──────────────────────────────────────────────────────────────────────────>

Request 1: addItem(apple)
│ EXCLUSIVE - acquires lock
│ ████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
│            │ completes, releases lock
│            │
Request 2: addItem(banana)
│            │ EXCLUSIVE - waits for lock...
│            │ ░░░░░░░░░░████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
│            │           │ acquires lock    │ completes
│            │           │                  │
Request 3: getContents()
│            │           │                  │ SHARED - waits for exclusive...
│            │           │                  │ ░░░░░░░░░█████░░░░░░░░░░░░░░░░░░░
│            │           │                  │         │ runs │
Request 4: getContents()
│            │           │                  │         │ SHARED - can run concurrent
│            │           │                  │         │ █████░░░░░░░░░░░░░░░░░░░
│                                                     │     │
│                                                     │ Both shared handlers
│                                                     │ run simultaneously

Legend: ████ = executing  ░░░░ = waiting
```

### State Storage

Virtual Object state is stored in Restate's embedded database (RocksDB):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STATE STORAGE MODEL                                       │
└─────────────────────────────────────────────────────────────────────────────┘

Restate Server Storage (RocksDB):

┌─────────────────────────────────────────────────────────────────────────────┐
│ Key                                    │ Value                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ state/ShoppingCart/user-123/items      │ [{"name":"apple","price":1.50}]    │
│ state/ShoppingCart/user-123/total      │ 1.50                               │
│ state/ShoppingCart/user-456/items      │ [{"name":"milk","price":3.00}]     │
│ state/ShoppingCart/user-456/total      │ 3.00                               │
└─────────────────────────────────────────────────────────────────────────────┘

State operations in handlers (Kotlin SDK):

ctx.get<T>(stateKey)       → Read from storage (journaled for replay)
ctx.set(stateKey, value)   → Write to storage (journaled, committed on success)
ctx.clear(stateKey)        → Delete from storage
ctx.clearAll()             → Delete all state for this object key
ctx.stateKeys()            → List all state keys for this object
```

### Temporal Workflow vs Restate Virtual Object: Complete Comparison

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// SAME ENTITY: Shopping Cart - implemented in both frameworks
// ═══════════════════════════════════════════════════════════════════════════


// ─────────────────────────────────────────────────────────────────────────────
// TEMPORAL VERSION
// ─────────────────────────────────────────────────────────────────────────────

@WorkflowInterface
interface ShoppingCartWorkflow {
    @WorkflowMethod
    fun checkout(): CheckoutResult  // Main workflow method

    @SignalMethod
    fun addItem(item: Item)

    @SignalMethod
    fun removeItem(itemId: String)

    @QueryMethod
    fun getContents(): CartContents
}

@WorkflowImpl
class ShoppingCartWorkflowImpl : ShoppingCartWorkflow {
    private val items = mutableListOf<Item>()
    private var checkedOut = false

    @SignalMethod
    override fun addItem(item: Item) {
        if (!checkedOut) {
            items.add(item)
        }
    }

    @SignalMethod
    override fun removeItem(itemId: String) {
        if (!checkedOut) {
            items.removeAll { it.id == itemId }
        }
    }

    @QueryMethod
    override fun getContents(): CartContents {
        return CartContents(items.toList(), items.sumOf { it.price })
    }

    override fun checkout(): CheckoutResult {
        // Wait for items or timeout
        Workflow.await(Duration.ofHours(24)) { items.isNotEmpty() }

        if (items.isEmpty()) {
            return CheckoutResult.ABANDONED
        }

        checkedOut = true
        val total = items.sumOf { it.price }
        val payment = activities.processPayment(total)

        return CheckoutResult.success(payment.id)
    }
}

// Client usage:
val cart = client.newWorkflowStub(ShoppingCartWorkflow::class.java, "cart-user-123")
WorkflowClient.start { cart.checkout() }  // Start workflow
cart.addItem(Item("apple", 1.50))          // Send signal
cart.addItem(Item("banana", 0.75))         // Send signal
val contents = cart.getContents()          // Query
// ... later, workflow completes checkout


// ─────────────────────────────────────────────────────────────────────────────
// RESTATE VERSION
// ─────────────────────────────────────────────────────────────────────────────

@VirtualObject
class ShoppingCart {

    @Handler
    suspend fun addItem(ctx: ObjectContext, item: Item) {
        val checkedOut = ctx.get<Boolean>(CHECKED_OUT) ?: false
        if (!checkedOut) {
            val items = ctx.get<List<Item>>(ITEMS) ?: emptyList()
            ctx.set(ITEMS, items + item)
        }
    }

    @Handler
    suspend fun removeItem(ctx: ObjectContext, itemId: String) {
        val checkedOut = ctx.get<Boolean>(CHECKED_OUT) ?: false
        if (!checkedOut) {
            val items = ctx.get<List<Item>>(ITEMS) ?: emptyList()
            ctx.set(ITEMS, items.filter { it.id != itemId })
        }
    }

    @Shared
    @Handler
    suspend fun getContents(ctx: SharedObjectContext): CartContents {
        val items = ctx.get<List<Item>>(ITEMS) ?: emptyList()
        return CartContents(items, items.sumOf { it.price })
    }

    @Handler
    suspend fun checkout(ctx: ObjectContext): CheckoutResult {
        val items = ctx.get<List<Item>>(ITEMS) ?: emptyList()

        if (items.isEmpty()) {
            return CheckoutResult.EMPTY
        }

        ctx.set(CHECKED_OUT, true)
        val total = items.sumOf { it.price }

        val payment = ctx.runBlock {
            paymentApi.processPayment(total)
        }

        // Clear cart
        ctx.clearAll()

        return CheckoutResult.success(payment.id)
    }

    companion object {
        private val ITEMS = StateKey.of<List<Item>>("items")
        private val CHECKED_OUT = StateKey.of<Boolean>("checkedOut")
    }
}

// Client usage:
restateClient.objectClient<ShoppingCart>("cart-user-123")
    .addItem(Item("apple", 1.50))

restateClient.objectClient<ShoppingCart>("cart-user-123")
    .addItem(Item("banana", 0.75))

val contents = restateClient.objectClient<ShoppingCart>("cart-user-123")
    .getContents()

val result = restateClient.objectClient<ShoppingCart>("cart-user-123")
    .checkout()
```

### Key Conceptual Differences

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           TEMPORAL WORKFLOW vs RESTATE VIRTUAL OBJECT                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────┬────────────────────────────────────────────┐
│       TEMPORAL WORKFLOW        │       RESTATE VIRTUAL OBJECT               │
├────────────────────────────────┼────────────────────────────────────────────┤
│                                │                                            │
│ LIFECYCLE:                     │ LIFECYCLE:                                 │
│ Has start and end              │ Exists as long as it has state            │
│ @WorkflowMethod runs once      │ Handlers can be called anytime            │
│                                │                                            │
│ IDENTITY:                      │ IDENTITY:                                  │
│ Workflow ID                    │ Object key (ctx.key())                    │
│                                │                                            │
│ INTERACTION:                   │ INTERACTION:                               │
│ Signals (@SignalMethod)        │ Handler calls (any @Handler)              │
│ Queries (@QueryMethod)         │ Shared handlers (@Shared)                 │
│                                │                                            │
│ STATE:                         │ STATE:                                     │
│ Class fields (via replay)      │ ctx.get/set (direct storage)              │
│                                │                                            │
│ CONCURRENCY:                   │ CONCURRENCY:                               │
│ Single-threaded by design      │ Exclusive handlers: one at a time         │
│ Signals queued                 │ Shared handlers: concurrent reads         │
│                                │                                            │
│ WAITING:                       │ WAITING:                                   │
│ Workflow.await { condition }   │ Awakeable or polling with sleep           │
│                                │                                            │
│ LONG-RUNNING:                  │ LONG-RUNNING:                              │
│ Continue-As-New needed         │ No limit (state persists)                 │
│                                │                                            │
└────────────────────────────────┴────────────────────────────────────────────┘
```

---

## 9. Signals vs Awakeables: Communication Patterns

### Temporal: Signals for External Events

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// TEMPORAL: Signals are the primary way to send data INTO a running workflow
// ═══════════════════════════════════════════════════════════════════════════

@WorkflowInterface
interface PaymentWorkflow {
    @WorkflowMethod
    fun processPayment(amount: Double): PaymentResult

    @SignalMethod
    fun onPaymentConfirmed(transactionId: String)

    @SignalMethod
    fun onPaymentFailed(reason: String)

    @QueryMethod
    fun getStatus(): String
}

@WorkflowImpl
class PaymentWorkflowImpl : PaymentWorkflow {
    private var status = "PENDING"
    private var transactionId: String? = null
    private var failureReason: String? = null

    @SignalMethod
    override fun onPaymentConfirmed(transactionId: String) {
        this.transactionId = transactionId
        this.status = "CONFIRMED"
    }

    @SignalMethod
    override fun onPaymentFailed(reason: String) {
        this.failureReason = reason
        this.status = "FAILED"
    }

    @QueryMethod
    override fun getStatus(): String = status

    override fun processPayment(amount: Double): PaymentResult {
        // Start external payment
        val paymentRef = activities.initiatePayment(amount)

        // Wait for signal (external system will call signal method)
        Workflow.await(Duration.ofMinutes(30)) {
            status == "CONFIRMED" || status == "FAILED"
        }

        return when (status) {
            "CONFIRMED" -> PaymentResult.success(transactionId!!)
            "FAILED" -> PaymentResult.failure(failureReason!!)
            else -> PaymentResult.timeout()
        }
    }
}

// External system (webhook handler) sends signal:
val workflow = client.newWorkflowStub(PaymentWorkflow::class.java, workflowId)
workflow.onPaymentConfirmed("txn_12345")  // This resumes the workflow
```

### Restate: Awakeables for External Events

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// RESTATE: Awakeables are promises that external systems can complete
// ═══════════════════════════════════════════════════════════════════════════

@VirtualObject
class PaymentProcessor {

    @Handler
    suspend fun processPayment(ctx: ObjectContext, amount: Double): PaymentResult {
        // Create an awakeable - a promise that can be completed externally
        val awakeable = ctx.awakeable<PaymentCallback>()

        // Start external payment, passing the awakeable ID
        ctx.runBlock {
            paymentGateway.initiatePayment(
                amount = amount,
                callbackId = awakeable.id  // External system uses this to complete
            )
        }

        // Store awakeable ID for the webhook handler
        ctx.set(AWAKEABLE_ID, awakeable.id)
        ctx.set(STATUS, "PENDING")

        // Suspend until awakeable is completed (handler exits, memory freed)
        val callback = awakeable.await()  // <-- SUSPENSION POINT

        // Resumed when external system completes the awakeable
        return when {
            callback.success -> {
                ctx.set(STATUS, "CONFIRMED")
                PaymentResult.success(callback.transactionId!!)
            }
            else -> {
                ctx.set(STATUS, "FAILED")
                PaymentResult.failure(callback.reason!!)
            }
        }
    }

    // Webhook handler - completes the awakeable
    @Handler
    suspend fun handleWebhook(ctx: ObjectContext, webhook: WebhookPayload) {
        val awakeableId = ctx.get<String>(AWAKEABLE_ID)
            ?: throw TerminalException("No pending payment")

        val callback = PaymentCallback(
            success = webhook.status == "SUCCESS",
            transactionId = webhook.transactionId,
            reason = webhook.errorMessage
        )

        // Complete the awakeable - this resumes processPayment
        ctx.awakeableHandle(awakeableId).resolve(callback)

        ctx.clear(AWAKEABLE_ID)
    }

    companion object {
        private val AWAKEABLE_ID = StateKey.of<String>("awakeableId")
        private val STATUS = StateKey.of<String>("status")
    }
}

// External webhook controller:
@PostMapping("/payment-webhook")
suspend fun handlePaymentWebhook(@RequestBody webhook: WebhookPayload) {
    restateClient
        .objectClient<PaymentProcessor>(webhook.paymentId)
        .handleWebhook(webhook)
}
```

### Alternative: Handler as Signal Equivalent

```kotlin
// ═══════════════════════════════════════════════════════════════════════════
// RESTATE: You can also use handlers + state as a signal-like pattern
// ═══════════════════════════════════════════════════════════════════════════

@VirtualObject
class OrderApproval {

    @Handler
    suspend fun submitForApproval(ctx: ObjectContext, order: Order): ApprovalResult {
        ctx.set(ORDER, order)
        ctx.set(STATUS, "PENDING")

        // Option 1: Use awakeable (suspends until completion)
        val awakeable = ctx.awakeable<ApprovalDecision>()
        ctx.set(AWAKEABLE_ID, awakeable.id)

        val decision = awakeable.await()
        return handleDecision(ctx, decision)
    }

    // This acts like a Temporal Signal
    @Handler
    suspend fun approve(ctx: ObjectContext, approver: String) {
        val awakeableId = ctx.get<String>(AWAKEABLE_ID)
        if (awakeableId != null) {
            ctx.awakeableHandle(awakeableId).resolve(
                ApprovalDecision(approved = true, approver = approver)
            )
        }
    }

    // This also acts like a Temporal Signal
    @Handler
    suspend fun reject(ctx: ObjectContext, reason: String) {
        val awakeableId = ctx.get<String>(AWAKEABLE_ID)
        if (awakeableId != null) {
            ctx.awakeableHandle(awakeableId).resolve(
                ApprovalDecision(approved = false, reason = reason)
            )
        }
    }

    @Shared
    @Handler
    suspend fun getStatus(ctx: SharedObjectContext): String {
        return ctx.get(STATUS) ?: "UNKNOWN"
    }

    private suspend fun handleDecision(
        ctx: ObjectContext,
        decision: ApprovalDecision
    ): ApprovalResult {
        ctx.clear(AWAKEABLE_ID)

        return if (decision.approved) {
            ctx.set(STATUS, "APPROVED")
            ctx.runBlock { processApprovedOrder(ctx.get(ORDER)!!) }
            ApprovalResult.approved(decision.approver!!)
        } else {
            ctx.set(STATUS, "REJECTED")
            ApprovalResult.rejected(decision.reason!!)
        }
    }

    companion object {
        private val ORDER = StateKey.of<Order>("order")
        private val STATUS = StateKey.of<String>("status")
        private val AWAKEABLE_ID = StateKey.of<String>("awakeableId")
    }
}
```

### Signals vs Awakeables Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               TEMPORAL SIGNALS vs RESTATE AWAKEABLES                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────┬────────────────────────────────────────────┐
│       TEMPORAL SIGNALS         │       RESTATE AWAKEABLES                   │
├────────────────────────────────┼────────────────────────────────────────────┤
│                                │                                            │
│ MECHANISM:                     │ MECHANISM:                                 │
│ @SignalMethod on workflow      │ ctx.awakeable<T>() returns promise + ID   │
│                                │                                            │
│ DELIVERY:                      │ DELIVERY:                                  │
│ Via workflow stub method call  │ Via awakeable ID (HTTP call to Restate)   │
│                                │                                            │
│ IDENTIFICATION:                │ IDENTIFICATION:                            │
│ Workflow ID required           │ Awakeable ID (opaque string)              │
│                                │                                            │
│ WAITING PATTERN:               │ WAITING PATTERN:                           │
│ Workflow.await { condition }   │ awakeable.await() suspends                │
│ (polls class field)            │                                            │
│                                │                                            │
│ MULTIPLE SIGNALS:              │ MULTIPLE AWAKEABLES:                       │
│ Different @SignalMethod each   │ Create multiple awakeables                │
│                                │                                            │
│ EXTERNAL COMPLETION:           │ EXTERNAL COMPLETION:                       │
│ stub.signalMethod(data)        │ POST to Restate with awakeable ID         │
│                                │ ctx.awakeableHandle(id).resolve(data)     │
│                                │                                            │
│ TYPED:                         │ TYPED:                                     │
│ Yes, method signature          │ Yes, generic type parameter               │
│                                │                                            │
└────────────────────────────────┴────────────────────────────────────────────┘
```

---

## 10. Service-to-Service Communication

### The Call Graph

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SERVICE CALL PATTERNS                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ OrderService.createOrder()                                                  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ // Synchronous call - waits for response                            │   │
│   │ val user = UserServiceClient.fromContext(ctx).getUser(userId).await()│   │
│   │                                                                     │   │
│   │ // Call to Virtual Object - addressed by key                        │   │
│   │ ShoppingCartClient.fromContext(ctx, cartId).checkout().await()      │   │
│   │                                                                     │   │
│   │ // Async call - fire and forget                                     │   │
│   │ NotificationServiceClient.fromContext(ctx).send().sendEmail(email)  │   │
│   │                                                                     │   │
│   │ // Delayed call - execute after delay                               │   │
│   │ ReminderServiceClient.fromContext(ctx)                              │   │
│   │   .send(Duration.ofHours(24))                                       │   │
│   │   .sendReminder(orderId)                                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                    │              │                │
                    ▼              ▼                ▼
            ┌───────────┐  ┌─────────────┐  ┌──────────────┐
            │UserService│  │ShoppingCart │  │Notification  │
            │           │  │ key=cartId  │  │   Service    │
            └───────────┘  └─────────────┘  └──────────────┘

All calls are:
• Journaled (durable)
• Automatically retried on failure
• Routed through Restate server
```

### Call Types in Kotlin

```kotlin
@Service
class OrderService {

    @Handler
    suspend fun process(ctx: Context, orderId: String): OrderResult {

        // ═══════════════════════════════════════════════════════════════════
        // 1. SYNCHRONOUS CALL (Request-Response)
        //    Caller waits for response. Journaled.
        // ═══════════════════════════════════════════════════════════════════
        val inventory = InventoryServiceClient.fromContext(ctx)
            .checkStock(orderId)
            .await()

        // ═══════════════════════════════════════════════════════════════════
        // 2. ASYNC CALL (Fire-and-Forget / Send)
        //    Returns immediately, doesn't wait for completion
        //    Still durable - will eventually execute
        // ═══════════════════════════════════════════════════════════════════
        AnalyticsServiceClient.fromContext(ctx)
            .send()
            .trackEvent(TrackingEvent("order_created", orderId))
        // Execution continues immediately, analytics runs in background

        // ═══════════════════════════════════════════════════════════════════
        // 3. DELAYED CALL
        //    Like async but with a delay before execution
        // ═══════════════════════════════════════════════════════════════════
        ReminderServiceClient.fromContext(ctx)
            .send(Duration.ofHours(1))  // 1 hour delay
            .sendAbandonedCartReminder(orderId)

        // ═══════════════════════════════════════════════════════════════════
        // 4. VIRTUAL OBJECT CALL
        //    Like service call but addressed by key
        // ═══════════════════════════════════════════════════════════════════
        PaymentObjectClient.fromContext(ctx, orderId)
            .processPayment(amount)
            .await()

        return OrderResult(status = "created")
    }
}
```

---

## 7. Failure Scenarios and Recovery

### Scenario 1: Service Crash Mid-Execution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CRASH RECOVERY SCENARIO                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Original Execution:
─────────────────────────────────────────────────────────────────────────────

@Handler
suspend fun process(ctx: Context, orderId: String): Result {
    val payment = ctx.runBlock { charge(orderId) }
    // Journal: [Input, Run{result:{id:"pay_123"}}]

    ctx.runBlock { updateInventory(orderId) }
    // Journal: [Input, Run{charge}, Run{inventory, result:{success:true}}]

    💥 CRASH HERE - process dies

    ctx.runBlock { sendEmail(orderId) }
    return Result(success = true)
}

Recovery (automatic):
─────────────────────────────────────────────────────────────────────────────

1. Restate detects invocation didn't complete (no Output entry in journal)
2. Restate re-invokes handler after timeout (configurable)
3. Handler replays:

@Handler
suspend fun process(ctx: Context, orderId: String): Result {
    val payment = ctx.runBlock { charge(orderId) }
    // ↑ Journal has this → returns {id:"pay_123"} without calling charge()

    ctx.runBlock { updateInventory(orderId) }
    // ↑ Journal has this → returns {success:true} without calling API

    // ← Journal ends here, now executing new code

    ctx.runBlock { sendEmail(orderId) }
    // ↑ Actually sends email, journals result

    return Result(success = true)
    // ↑ Journals Output, invocation complete
}

Result: Email sent exactly once, no double charge, no double inventory update
```

### Scenario 2: Side Effect Failure with Retry

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RETRY BEHAVIOR                                            │
└─────────────────────────────────────────────────────────────────────────────┘

val result = ctx.runBlock {
    val response = httpClient.post("https://api.example.com/data")
    if (!response.isSuccessful) {
        throw RuntimeException("API failed")  // Retriable error
    }
    response.body()
}

What happens on error:

Attempt 1: httpClient.post() → 500 error → throws RuntimeException
           Restate retries (exponential backoff)

Attempt 2: httpClient.post() → 500 error → throws RuntimeException
           Restate retries

Attempt 3: httpClient.post() → 200 OK → returns data
           Result journaled: {data: ...}

Subsequent replays: Journal has result, no HTTP call made

IMPORTANT: Retries are WITHIN a single invocation attempt
           The journal entry is only written on SUCCESS
```

### Scenario 3: Terminal Error (No Retry)

```kotlin
val result = ctx.runBlock {
    val isValid = validateInput(input)

    if (!isValid) {
        // Terminal error - STOP retrying, fail permanently
        throw TerminalException("Invalid input")
    }

    process(input)
}

// TerminalException:
// - Stops retry loop immediately
// - Fails the invocation
// - Error is returned to caller
// - Journal records the failure
```

---

## 8. Practical Examples

### Example 1: Order Processing Saga

```kotlin
@Service
class OrderService {

    @Handler
    suspend fun processOrder(ctx: Context, order: Order): OrderResult {
        // Step 1: Reserve inventory
        val reservation = ctx.runBlock {
            inventoryApi.reserve(order.items)
        }

        // Step 2: Charge payment
        val payment = try {
            ctx.runBlock {
                paymentApi.charge(order.userId, order.total)
            }
        } catch (e: Exception) {
            // Compensation: Release inventory if payment fails
            ctx.runBlock {
                inventoryApi.release(reservation.id)
            }
            throw TerminalException("Payment failed: ${e.message}")
        }

        // Step 3: Create shipment
        val shipment = try {
            ctx.runBlock {
                shipmentApi.create(order.address, reservation.id)
            }
        } catch (e: Exception) {
            // Compensation: Refund and release inventory
            ctx.runBlock { paymentApi.refund(payment.id) }
            ctx.runBlock { inventoryApi.release(reservation.id) }
            throw TerminalException("Shipment failed: ${e.message}")
        }

        // Step 4: Send confirmation (async - fire and forget)
        NotificationServiceClient.fromContext(ctx)
            .send()
            .sendOrderConfirmation(OrderConfirmation(order.id, shipment.id))

        return OrderResult(
            orderId = order.id,
            paymentId = payment.id,
            shipmentId = shipment.id,
            status = "CONFIRMED"
        )
    }
}
```

### Example 2: Stateful User Session (Virtual Object)

```kotlin
@VirtualObject
class UserSession {

    @Handler
    suspend fun login(ctx: ObjectContext, credentials: Credentials): LoginResult {
        val existingSession = ctx.get<Session>(SESSION)
        if (existingSession != null && !existingSession.expired) {
            return LoginResult(success = false, reason = "Already logged in")
        }

        val authResult = ctx.runBlock {
            authService.verify(credentials)
        }

        if (!authResult.valid) {
            return LoginResult(success = false, reason = "Invalid credentials")
        }

        val session = Session(
            userId = ctx.key(),  // Virtual object key
            token = authResult.token,
            loginTime = System.currentTimeMillis(),
            expired = false
        )

        ctx.set(SESSION, session)

        // Schedule session expiry (30 minutes)
        UserSessionClient.fromContext(ctx, ctx.key())
            .send(Duration.ofMinutes(30))
            .expireSession()

        return LoginResult(success = true, token = session.token)
    }

    @Handler
    suspend fun logout(ctx: ObjectContext): LogoutResult {
        ctx.clear(SESSION)
        return LogoutResult(success = true)
    }

    @Handler
    suspend fun expireSession(ctx: ObjectContext) {
        val session = ctx.get<Session>(SESSION)
        if (session != null) {
            ctx.set(SESSION, session.copy(expired = true))
        }
    }

    @Shared
    @Handler
    suspend fun getSession(ctx: SharedObjectContext): Session? {
        return ctx.get(SESSION)
    }

    companion object {
        private val SESSION = StateKey.of<Session>("session")
    }
}
```

### Example 3: External Event Handling (Awakeable)

```kotlin
@VirtualObject
class PaymentProcessor {

    @Handler
    suspend fun initiatePayment(ctx: ObjectContext, amount: Double): PaymentResult {
        // Create awakeable - external systems can complete this
        val awakeable = ctx.awakeable<PaymentCallback>()

        // Start external payment flow
        ctx.runBlock {
            paymentGateway.initiatePayment(
                paymentId = ctx.key(),
                amount = amount,
                callbackId = awakeable.id  // Gateway will call back with this ID
            )
        }

        ctx.set(STATUS, "PENDING")
        ctx.set(AWAKEABLE_ID, awakeable.id)

        // Wait for external callback (handler suspends here)
        val callback = awakeable.await()

        // Resumed after external system completed the awakeable
        return if (callback.success) {
            ctx.set(STATUS, "COMPLETED")
            ctx.set(TRANSACTION_ID, callback.transactionId)
            PaymentResult(success = true, transactionId = callback.transactionId)
        } else {
            ctx.set(STATUS, "FAILED")
            ctx.set(ERROR, callback.error)
            PaymentResult(success = false, error = callback.error)
        }
    }

    // Called by webhook handler when payment gateway responds
    @Handler
    suspend fun handleWebhook(ctx: ObjectContext, webhook: WebhookPayload) {
        val awakeableId = ctx.get<String>(AWAKEABLE_ID)
            ?: throw TerminalException("No pending payment")

        // Complete the awakeable - this resumes initiatePayment
        val callback = if (webhook.status == "SUCCESS") {
            PaymentCallback(success = true, transactionId = webhook.transactionId)
        } else {
            PaymentCallback(success = false, error = webhook.errorMessage)
        }

        ctx.awakeableHandle(awakeableId).resolve(callback)
    }

    companion object {
        private val STATUS = StateKey.of<String>("status")
        private val AWAKEABLE_ID = StateKey.of<String>("awakeableId")
        private val TRANSACTION_ID = StateKey.of<String>("transactionId")
        private val ERROR = StateKey.of<String>("error")
    }
}
```

---

## Summary: Mental Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESTATE MENTAL MODEL                                      │
└─────────────────────────────────────────────────────────────────────────────┘

1. YOUR CODE is a regular HTTP handler
   - No special runtime, just HTTP endpoints (Spring Boot compatible)
   - Stateless - all state lives in Restate

2. RESTATE is a proxy that adds durability
   - Sits between clients and your services
   - Stores journals and state
   - Handles retries, replay, and recovery

3. THE JOURNAL is your execution checkpoint
   - Records results of ctx.* operations
   - On replay, returns journaled results without re-execution
   - Only ctx.* call sequence must be deterministic

4. ctx.runBlock { } = durable side effect
   - Block executes once
   - Result is journaled
   - On replay, returns result from journal

5. VIRTUAL OBJECTS = keyed stateful entities
   - Persistent state via ctx.get/set
   - Concurrency control via exclusive handlers
   - Like actors or aggregates

6. REPLAY happens on:
   - Service crash recovery
   - After ctx.sleep() timer fires
   - After awakeable completion
   - Network reconnection

Key principle: Make side effects idempotent OR wrap them in ctx.runBlock { }
              Everything else "just works"
```