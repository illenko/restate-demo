# Temporal vs Restate: Concept Mapping Guide

> A comprehensive guide for understanding how Temporal concepts map to Restate, designed for developers migrating between frameworks or learning both.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Core Concepts Mapping](#2-core-concepts-mapping)
3. [Execution Model](#3-execution-model)
4. [State Management](#4-state-management)
5. [Durability & Replay](#5-durability--replay)
6. [Activities & Side Effects](#6-activities--side-effects)
7. [Error Handling & Retries](#7-error-handling--retries)
8. [Communication Patterns](#8-communication-patterns)
9. [Timers & Scheduling](#9-timers--scheduling)
10. [Child Workflows / Sub-Workflows](#10-child-workflows--sub-workflows)
11. [Querying State](#11-querying-state)
12. [Versioning & Deployments](#12-versioning--deployments)
13. [Code Pattern Comparisons](#13-code-pattern-comparisons)
14. [Key Questions for AI Agent](#14-key-questions-for-ai-agent)

---

## 1. Architecture Overview

### Temporal Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TEMPORAL ARCHITECTURE                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────────────────┐         ┌──────────────┐
│              │         │     TEMPORAL SERVER      │         │              │
│    CLIENT    │────────▶│                          │◀────────│    WORKER    │
│  (API/App)   │         │  • History storage       │  polls  │              │
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

Key characteristics:
• Separate server component (self-hosted or Temporal Cloud)
• Workers poll for tasks
• Event sourcing (history-based replay)
• Explicit workflow/activity separation
```

### Restate Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           RESTATE ARCHITECTURE                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────────────────┐         ┌──────────────┐
│              │         │      RESTATE SERVER      │         │              │
│    CLIENT    │────────▶│                          │────────▶│   SERVICE    │
│  (API/App)   │   HTTP  │  • Journal storage       │   HTTP  │   HANDLER    │
│              │         │  • State management      │  invoke │              │
│  Invokes     │         │  • Durable execution     │         │  • Handlers  │
│  handlers    │         │  • Virtual objects       │         │  • Objects   │
└──────────────┘         └──────────────────────────┘         └──────────────┘
                                     │
                                     ▼
                         ┌──────────────────────────┐
                         │   EMBEDDED STORAGE       │
                         │   (RocksDB) or External  │
                         └──────────────────────────┘

Key characteristics:
• Server acts as proxy/router
• Push-based invocation (HTTP calls to services)
• Journal-based replay (not event sourcing)
• Services are just HTTP endpoints with SDK
```

### Side-by-Side Comparison

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **Server role** | Orchestrator + storage | Proxy + durable execution engine |
| **Worker model** | Pull (workers poll) | Push (server invokes handlers) |
| **Communication** | gRPC (task queues) | HTTP/gRPC |
| **State storage** | External DB required | Embedded (RocksDB) or external |
| **Deployment** | Server + Workers + DB | Server + Services |

---

## 2. Core Concepts Mapping

### Terminology Mapping

| Temporal Concept | Restate Equivalent | Notes |
|------------------|-------------------|-------|
| **Workflow** | **Service Handler** or **Virtual Object** | Restate has multiple patterns |
| **Activity** | **Side effect** / **External call** | Via `ctx.run()` or `ctx.sideEffect()` |
| **Worker** | **Service** (HTTP endpoint) | Services register handlers |
| **Task Queue** | **Service endpoint** | Routing is URL-based |
| **Workflow ID** | **Invocation ID** or **Virtual Object Key** | Depends on pattern used |
| **Signal** | **Handler call** on Virtual Object | Different mechanism |
| **Query** | **Shared handler** on Virtual Object | Read-only access to state |
| **Child Workflow** | **Service-to-service call** | Via `ctx.serviceClient()` |
| **Continue-As-New** | Not needed (different model) | Restate handles differently |
| **Timer** | `ctx.sleep()` | Similar concept |

### Concept Deep Dive

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TEMPORAL WORKFLOW vs RESTATE PATTERNS                     │
└─────────────────────────────────────────────────────────────────────────────┘

TEMPORAL WORKFLOW:
──────────────────
• Long-running, stateful execution
• Identified by Workflow ID
• Has methods: @WorkflowMethod, @SignalMethod, @QueryMethod
• State maintained via replay of event history
• Explicit workflow definition required

RESTATE OPTIONS:
────────────────

Option 1: SERVICE (Stateless Handler)
├── Simple request-response
├── Durable execution with journal
├── No built-in state between calls
└── Good for: Orchestration, sagas

Option 2: VIRTUAL OBJECT (Stateful Entity)
├── Keyed by unique ID (like Workflow ID)
├── Has persistent state via ctx.state
├── Exclusive handler = one execution at a time per key
├── Shared handler = concurrent reads (like Query)
└── Good for: Entities, aggregates, stateful workflows

Option 3: WORKFLOW (Restate's workflow primitive)
├── Similar to Temporal workflow
├── Has run() method + signal handlers
├── Durable promise for result
└── Good for: Long-running processes
```

---

## 3. Execution Model

### Temporal Execution Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TEMPORAL EXECUTION MODEL                                  │
└─────────────────────────────────────────────────────────────────────────────┘

1. Client starts workflow
        │
        ▼
2. Temporal Server creates Workflow Task
        │
        ▼
3. Worker POLLS and picks up task
        │
        ▼
4. Worker executes workflow code
        │
        ├──► Hits activity call
        │         │
        │         ▼
        │    Records "ActivityScheduled" command
        │         │
        │         ▼
        │    Returns control to Temporal
        │         │
        │         ▼
        │    Temporal creates Activity Task
        │         │
        │         ▼
        │    Worker (any) picks up and executes
        │         │
        │         ▼
        │    Result saved to history
        │         │
        │         ▼
        │    New Workflow Task created
        │         │
        │         ▼
        └──► Worker picks up, REPLAYS from history
                  │
                  ▼
             Continues execution

Key: Event sourcing - full history replay on each workflow task
```

### Restate Execution Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESTATE EXECUTION MODEL                                   │
└─────────────────────────────────────────────────────────────────────────────┘

1. Client invokes handler (HTTP request)
        │
        ▼
2. Restate Server receives request
        │
        ▼
3. Restate INVOKES service handler (HTTP call)
        │
        ▼
4. Handler executes code
        │
        ├──► Hits ctx.run() (side effect)
        │         │
        │         ▼
        │    Restate journals the intent
        │         │
        │         ▼
        │    Code executes, result journaled
        │         │
        │         ▼
        │    Continues to next step
        │
        ├──► Hits ctx.sleep() (timer)
        │         │
        │         ▼
        │    Restate suspends handler
        │         │
        │         ▼
        │    Timer fires, Restate re-invokes handler
        │         │
        │         ▼
        │    REPLAY from journal (only journaled steps)
        │         │
        │         ▼
        └──► Continues after sleep

Key: Journal-based - only replay journaled operations, not full history
```

### Key Differences

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **Invocation** | Worker polls for tasks | Server pushes to service |
| **Replay trigger** | Every workflow task | Only after suspension/crash |
| **Replay scope** | Full event history | Journal entries only |
| **Determinism requirement** | Very strict (all code) | Only journaled operations |
| **Code between activities** | Must be deterministic | Can be non-deterministic |

---

## 4. State Management

### Temporal State Management

```kotlin
// TEMPORAL: State via workflow fields + replay

@WorkflowImpl
class PaymentWorkflowImpl : PaymentWorkflow {
    // State is maintained via replay
    private var status = "PENDING"
    private var processedCount = 0
    private val results = mutableListOf<String>()

    override fun process(payments: List<String>): Result {
        payments.forEach { payment ->
            val result = activities.processPayment(payment)  // Recorded in history
            results.add(result)
            processedCount++
            status = "PROCESSING"
        }
        status = "COMPLETED"
        return Result(results)
    }

    @QueryMethod
    override fun getStatus(): String = status  // Returns current state
}

// State reconstruction:
// - On replay, workflow code re-executes
// - Activity calls return results from history
// - Variables get same values as original execution
```

### Restate State Management

```typescript
// RESTATE: Explicit state API with Virtual Objects

const paymentObject = restate.object({
  name: "payment",
  handlers: {
    // Exclusive handler - one at a time per key
    process: async (ctx: ObjectContext, payments: string[]) => {
      // Explicit state access
      let status = await ctx.get<string>("status") ?? "PENDING";
      let processedCount = await ctx.get<number>("processedCount") ?? 0;
      let results = await ctx.get<string[]>("results") ?? [];

      for (const payment of payments) {
        // Side effect - journaled
        const result = await ctx.run(() => processPayment(payment));
        results.push(result);
        processedCount++;

        // Explicit state update
        ctx.set("results", results);
        ctx.set("processedCount", processedCount);
        ctx.set("status", "PROCESSING");
      }

      ctx.set("status", "COMPLETED");
      return { results };
    },

    // Shared handler - concurrent reads allowed (like Query)
    getStatus: async (ctx: ObjectSharedContext) => {
      return await ctx.get<string>("status");
    }
  }
});

// State storage:
// - Explicitly stored via ctx.set()
// - Persisted in Restate's storage
// - Available across invocations
// - No replay needed to access state
```

### State Comparison

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **State storage** | Implicit (via replay) | Explicit (ctx.get/set) |
| **State access** | Reconstruct via replay | Direct read from storage |
| **State persistence** | Event history | Key-value store |
| **Cross-invocation** | Via signals/queries | Via Virtual Object state |
| **State size limit** | History size limit | Per-key storage limit |

---

## 5. Durability & Replay

### Temporal Durability

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TEMPORAL DURABILITY MODEL                                 │
└─────────────────────────────────────────────────────────────────────────────┘

Event History (append-only log):
────────────────────────────────
Event 1:  WorkflowExecutionStarted
Event 2:  WorkflowTaskScheduled
Event 3:  WorkflowTaskStarted
Event 4:  WorkflowTaskCompleted
Event 5:  ActivityTaskScheduled {type: "processPayment", input: "pay_001"}
Event 6:  ActivityTaskStarted
Event 7:  ActivityTaskCompleted {result: "success"}
Event 8:  ActivityTaskScheduled {type: "processPayment", input: "pay_002"}
...

Replay mechanism:
────────────────
1. Worker receives Workflow Task
2. Loads FULL event history from server
3. Re-executes workflow code from beginning
4. SDK matches each activity call to history event
5. Returns historical result (no re-execution)
6. Continues until caught up with history
7. Executes new code, generating new commands

CRITICAL: All workflow code must be deterministic
         Same code + same history = same execution path
```

### Restate Durability

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESTATE DURABILITY MODEL                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Journal (per invocation):
─────────────────────────
Entry 1: InputMessage {payments: ["pay_001", "pay_002"]}
Entry 2: Run {closure_id: 1, result: "success"}
Entry 3: SetState {key: "processedCount", value: 1}
Entry 4: Run {closure_id: 2, result: "success"}
Entry 5: SetState {key: "processedCount", value: 2}
Entry 6: Sleep {wake_up_time: ...}
Entry 7: OutputMessage {result: {...}}

Replay mechanism:
────────────────
1. Handler invoked (or re-invoked after suspend)
2. Restate provides journal to SDK
3. SDK replays ONLY journaled operations:
   - ctx.run() → return journaled result
   - ctx.get() → return journaled value
   - ctx.sleep() → skip if timer fired
4. Non-journaled code executes normally
5. After journal exhausted, new operations journaled

DIFFERENCE: Only ctx.* operations must be deterministic
            Regular code between them can vary
```

### Determinism Requirements Comparison

| Code Type | Temporal | Restate |
|-----------|----------|---------|
| Activity/side effect calls | Must match history order | Must match journal order |
| Code between activities | Must be deterministic | Can be non-deterministic |
| Random/UUID generation | Use Workflow.random() | Can use regular random (outside ctx.run) |
| Current time | Use Workflow.currentTimeMillis() | Can use Date.now() (outside ctx.run) |
| HashMap iteration | Must be deterministic | Only deterministic inside ctx.run |

---

## 6. Activities & Side Effects

### Temporal Activities

```kotlin
// TEMPORAL: Explicit Activity interface + implementation

// 1. Define interface
@ActivityInterface
interface PaymentActivities {
    @ActivityMethod
    fun processPayment(paymentId: String): PaymentResult

    @ActivityMethod
    fun sendNotification(userId: String, message: String)
}

// 2. Implement
class PaymentActivitiesImpl : PaymentActivities {
    override fun processPayment(paymentId: String): PaymentResult {
        // Real I/O here - HTTP calls, DB, etc.
        return httpClient.post("/payments/$paymentId/process")
    }

    override fun sendNotification(userId: String, message: String) {
        emailService.send(userId, message)
    }
}

// 3. Register with worker
worker.registerActivitiesImplementations(PaymentActivitiesImpl())

// 4. Use in workflow
@WorkflowImpl
class PaymentWorkflowImpl : PaymentWorkflow {
    private val activities = Workflow.newActivityStub(
        PaymentActivities::class.java,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofSeconds(30))
            .setRetryOptions(RetryOptions.newBuilder()
                .setMaximumAttempts(3)
                .build())
            .build()
    )

    override fun process(paymentId: String): Result {
        val result = activities.processPayment(paymentId)  // Durable call
        activities.sendNotification(userId, "Payment processed")
        return result
    }
}
```

### Restate Side Effects

```typescript
// RESTATE: ctx.run() for side effects

const paymentService = restate.service({
  name: "payment",
  handlers: {
    process: async (ctx: Context, paymentId: string) => {
      // Side effect - automatically retried, result journaled
      const result = await ctx.run("process-payment", async () => {
        // Real I/O here
        const response = await fetch(`/payments/${paymentId}/process`, {
          method: 'POST'
        });
        return response.json();
      });

      // Another side effect
      await ctx.run("send-notification", async () => {
        await emailService.send(userId, "Payment processed");
      });

      return result;
    }
  }
});

// Key differences:
// - No separate interface/implementation
// - Side effects are inline closures
// - Automatic retry with configurable policy
// - Result automatically journaled
```

### Comparison

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **Definition** | Separate interface + impl | Inline closure |
| **Registration** | Worker registration | Part of handler code |
| **Retry config** | ActivityOptions | ctx.run options or global |
| **Execution** | On any worker | In same service process |
| **Distribution** | Across workers via task queue | Same process (or service call) |

---

## 7. Error Handling & Retries

### Temporal Error Handling

```kotlin
// TEMPORAL: Retry policies on activities

// Activity-level retry
val activityOptions = ActivityOptions.newBuilder()
    .setStartToCloseTimeout(Duration.ofSeconds(30))
    .setRetryOptions(RetryOptions.newBuilder()
        .setInitialInterval(Duration.ofSeconds(1))
        .setBackoffCoefficient(2.0)
        .setMaximumInterval(Duration.ofSeconds(60))
        .setMaximumAttempts(5)
        .setDoNotRetry(listOf(
            IllegalArgumentException::class.java.name  // Don't retry these
        ))
        .build())
    .build()

// In workflow - catching activity failures
override fun process(paymentId: String): Result {
    try {
        return activities.processPayment(paymentId)
    } catch (e: ActivityFailure) {
        // Activity failed after all retries
        when (val cause = e.cause) {
            is ApplicationFailure -> {
                // Handle specific failure
                logger.error("Payment failed: ${cause.message}")
                return Result.failed(cause.message)
            }
            else -> throw e  // Re-throw unexpected errors
        }
    }
}

// Workflow-level error handling
// - Unhandled exceptions fail the workflow
// - Can be caught by parent workflow
// - Workflow can be retried via WorkflowOptions
```

### Restate Error Handling

```typescript
// RESTATE: Automatic retries with terminal errors

const paymentService = restate.service({
  name: "payment",
  handlers: {
    process: async (ctx: Context, paymentId: string) => {
      try {
        // Automatic infinite retry by default
        const result = await ctx.run("process-payment", async () => {
          const response = await fetch(`/payments/${paymentId}/process`);
          if (!response.ok) {
            if (response.status === 400) {
              // Terminal error - don't retry
              throw new restate.TerminalError("Invalid payment", {
                errorCode: "INVALID_PAYMENT"
              });
            }
            // Retriable error - will be retried
            throw new Error("Payment service unavailable");
          }
          return response.json();
        });

        return result;

      } catch (e) {
        if (e instanceof restate.TerminalError) {
          // Terminal errors propagate to caller
          return { status: "failed", error: e.message };
        }
        throw e;  // Other errors cause retry
      }
    }
  }
});

// Key concepts:
// - TerminalError: Stop retrying, fail permanently
// - Regular Error: Keep retrying (infinite by default)
// - Retry policy configurable globally or per-operation
```

### Comparison

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **Default retry** | Configured per activity | Infinite retry |
| **Stop retry** | setDoNotRetry + exception types | TerminalError |
| **Retry config** | Per activity stub | Global or per ctx.run |
| **Backoff** | Exponential (configurable) | Exponential (configurable) |
| **Max attempts** | Configurable | Infinite unless specified |
| **Timeout** | startToClose, scheduleToClose | Per-operation timeout |

---

## 8. Communication Patterns

### Temporal Signals

```kotlin
// TEMPORAL: Signals for async communication

@WorkflowInterface
interface OrderWorkflow {
    @WorkflowMethod
    fun processOrder(orderId: String): OrderResult

    @SignalMethod
    fun approveOrder(approver: String)

    @SignalMethod
    fun cancelOrder(reason: String)
}

@WorkflowImpl
class OrderWorkflowImpl : OrderWorkflow {
    private var approved = false
    private var cancelled = false

    @SignalMethod
    override fun approveOrder(approver: String) {
        approved = true
    }

    @SignalMethod
    override fun cancelOrder(reason: String) {
        cancelled = true
    }

    override fun processOrder(orderId: String): OrderResult {
        // Wait for signal
        Workflow.await { approved || cancelled }

        return if (approved) {
            activities.fulfillOrder(orderId)
            OrderResult.COMPLETED
        } else {
            OrderResult.CANCELLED
        }
    }
}

// Client sends signal:
val workflow = workflowClient.newWorkflowStub(OrderWorkflow::class.java, workflowId)
workflow.approveOrder("manager-1")
```

### Restate Virtual Object Handlers

```typescript
// RESTATE: Handler calls on Virtual Objects

const orderObject = restate.object({
  name: "order",
  handlers: {
    // Main processing (exclusive - one at a time)
    processOrder: async (ctx: ObjectContext, orderId: string) => {
      // Wait for approval by checking state
      const approved = await ctx.run("wait-approval", async () => {
        // Poll or use awakeable
        while (true) {
          const status = await ctx.get<string>("approval_status");
          if (status === "approved") return true;
          if (status === "cancelled") return false;
          await ctx.sleep(1000);  // Check again
        }
      });

      if (approved) {
        await ctx.run("fulfill", () => fulfillOrder(orderId));
        return { status: "COMPLETED" };
      }
      return { status: "CANCELLED" };
    },

    // Signal equivalent - another handler on same object
    approveOrder: async (ctx: ObjectContext, approver: string) => {
      ctx.set("approval_status", "approved");
      ctx.set("approved_by", approver);
    },

    cancelOrder: async (ctx: ObjectContext, reason: string) => {
      ctx.set("approval_status", "cancelled");
      ctx.set("cancel_reason", reason);
    }
  }
});

// Client sends "signal":
await restate.objectClient("order", orderId).approveOrder("manager-1");
```

### Better Pattern: Awakeables

```typescript
// RESTATE: Awakeables for waiting on external events

const orderObject = restate.object({
  name: "order",
  handlers: {
    processOrder: async (ctx: ObjectContext, orderId: string) => {
      // Create awakeable - returns ID to share externally
      const awakeable = ctx.awakeable<ApprovalResult>();

      // Store awakeable ID so external system can complete it
      ctx.set("pending_awakeable", awakeable.id);

      // Suspend until awakeable is completed
      const approval = await awakeable.promise;

      if (approval.approved) {
        await ctx.run("fulfill", () => fulfillOrder(orderId));
        return { status: "COMPLETED" };
      }
      return { status: "CANCELLED" };
    },

    // Handler to complete the awakeable
    completeApproval: async (ctx: ObjectContext, result: ApprovalResult) => {
      const awakeableId = await ctx.get<string>("pending_awakeable");
      ctx.resolveAwakeable(awakeableId, result);
    }
  }
});
```

---

## 9. Timers & Scheduling

### Temporal Timers

```kotlin
// TEMPORAL: Workflow.sleep and Workflow.await

override fun processWithTimeout(orderId: String): Result {
    // Simple sleep
    Workflow.sleep(Duration.ofMinutes(5))

    // Await with timeout
    val approved = Workflow.await(Duration.ofHours(24)) {
        approvalReceived
    }

    if (!approved) {
        // Timeout occurred
        return Result.TIMEOUT
    }

    // Await without timeout (wait forever)
    Workflow.await { paymentReceived }

    return Result.SUCCESS
}

// Timer events are recorded in history
// On replay, timers that already fired are skipped
```

### Restate Timers

```typescript
// RESTATE: ctx.sleep

const orderService = restate.service({
  name: "order",
  handlers: {
    processWithTimeout: async (ctx: Context, orderId: string) => {
      // Simple sleep
      await ctx.sleep(5 * 60 * 1000);  // 5 minutes in ms

      // Sleep with awakeable for "await with timeout" pattern
      const awakeable = ctx.awakeable<boolean>();

      // Store for external completion
      await ctx.run("store-awakeable", async () => {
        await storeAwakeableId(orderId, awakeable.id);
      });

      // Race between timer and awakeable
      const result = await Promise.race([
        awakeable.promise,
        ctx.sleep(24 * 60 * 60 * 1000).then(() => null)  // 24 hours
      ]);

      if (result === null) {
        return { status: "TIMEOUT" };
      }

      return { status: "SUCCESS" };
    }
  }
});

// Timers are journaled
// On replay, completed timers return immediately
```

### Comparison

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **Sleep** | `Workflow.sleep(Duration)` | `ctx.sleep(ms)` |
| **Await condition** | `Workflow.await { condition }` | Awakeable + polling |
| **Await with timeout** | `Workflow.await(duration) { }` | `Promise.race` with sleep |
| **Timer persistence** | Event in history | Journal entry |
| **On replay** | Skip if timer fired | Skip if timer fired |

---

## 10. Child Workflows / Sub-Workflows

### Temporal Child Workflows

```kotlin
// TEMPORAL: Child workflow invocation

@WorkflowImpl
class ParentWorkflowImpl : ParentWorkflow {
    override fun process(gateways: List<String>): Result {
        val childOptions = ChildWorkflowOptions.newBuilder()
            .setWorkflowId("child-${Workflow.getInfo().workflowId}-$gateway")
            .setTaskQueue("gateway-processing")
            .build()

        // Start child workflows in parallel
        val promises = gateways.map { gateway ->
            val child = Workflow.newChildWorkflowStub(
                GatewayWorkflow::class.java,
                childOptions
            )
            Async.function { child.processGateway(gateway) }
        }

        // Wait for all children
        val results = promises.map { it.get() }

        return aggregateResults(results)
    }
}

// Child workflow has its own:
// - Workflow ID
// - Event history
// - Lifecycle (can continue after parent completes with ParentClosePolicy)
```

### Restate Service Calls

```typescript
// RESTATE: Service-to-service calls

const parentService = restate.service({
  name: "parent",
  handlers: {
    process: async (ctx: Context, gateways: string[]) => {
      // Call other services (like child workflows)
      const promises = gateways.map(gateway =>
        ctx.serviceClient(gatewayService)
          .processGateway(gateway)
      );

      // Wait for all (parallel execution)
      const results = await Promise.all(promises);

      return aggregateResults(results);
    }
  }
});

const gatewayService = restate.service({
  name: "gateway",
  handlers: {
    processGateway: async (ctx: Context, gateway: string) => {
      // Process gateway...
      return result;
    }
  }
});

// Service calls are:
// - Durable (journaled)
// - Automatically retried
// - Can be to same or different service
```

### Virtual Object as Child

```typescript
// RESTATE: Virtual Objects for entity-like children

const gatewayObject = restate.object({
  name: "gateway-processor",
  handlers: {
    process: async (ctx: ObjectContext, payments: string[]) => {
      // ctx.key is the gateway name
      const gateway = ctx.key;

      // Has its own state
      ctx.set("status", "processing");
      ctx.set("processed_count", 0);

      // Process...
      return result;
    },

    getStatus: async (ctx: ObjectSharedContext) => {
      return {
        gateway: ctx.key,
        status: await ctx.get("status"),
        count: await ctx.get("processed_count")
      };
    }
  }
});

// Call from parent:
const results = await Promise.all(
  gateways.map(gateway =>
    ctx.objectClient(gatewayObject, gateway).process(payments)
  )
);
```

---

## 11. Querying State

### Temporal Queries

```kotlin
// TEMPORAL: @QueryMethod for synchronous state access

@WorkflowInterface
interface PaymentWorkflow {
    @WorkflowMethod
    fun process(payments: List<String>): Result

    @QueryMethod
    fun getProgress(): ProgressInfo

    @QueryMethod
    fun getFailedPayments(): List<String>
}

@WorkflowImpl
class PaymentWorkflowImpl : PaymentWorkflow {
    private var processed = 0
    private var total = 0
    private val failed = mutableListOf<String>()

    @QueryMethod
    override fun getProgress(): ProgressInfo {
        return ProgressInfo(processed, total, failed.size)
    }

    @QueryMethod
    override fun getFailedPayments(): List<String> {
        return failed.toList()
    }

    override fun process(payments: List<String>): Result {
        total = payments.size
        // ... processing logic
    }
}

// Client queries:
val stub = workflowClient.newWorkflowStub(PaymentWorkflow::class.java, workflowId)
val progress = stub.getProgress()  // Synchronous, read-only

// Query characteristics:
// - Executed on worker with cached workflow
// - May require replay to rebuild state
// - Read-only (cannot modify state)
// - No history events created
```

### Restate Shared Handlers

```typescript
// RESTATE: Shared handlers on Virtual Objects

const paymentObject = restate.object({
  name: "payment-processor",
  handlers: {
    // Exclusive handler - main processing
    process: async (ctx: ObjectContext, payments: string[]) => {
      ctx.set("total", payments.length);
      ctx.set("processed", 0);
      ctx.set("failed", []);

      // ... processing logic

      ctx.set("status", "completed");
      return result;
    },

    // Shared handler - concurrent reads (like Query)
    getProgress: restate.handlers.object.shared(
      async (ctx: ObjectSharedContext): Promise<ProgressInfo> => {
        return {
          processed: await ctx.get<number>("processed") ?? 0,
          total: await ctx.get<number>("total") ?? 0,
          failedCount: (await ctx.get<string[]>("failed"))?.length ?? 0
        };
      }
    ),

    // Another shared handler
    getFailedPayments: restate.handlers.object.shared(
      async (ctx: ObjectSharedContext): Promise<string[]> => {
        return await ctx.get<string[]>("failed") ?? [];
      }
    )
  }
});

// Client queries:
const progress = await restate
  .objectClient("payment-processor", workflowId)
  .getProgress();

// Shared handler characteristics:
// - Can run concurrently with other shared handlers
// - Cannot run concurrently with exclusive handlers
// - Direct state access (no replay needed)
// - Read-only operations
```

### Comparison

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **Mechanism** | @QueryMethod | Shared handler |
| **State access** | Via replay/cache | Direct from storage |
| **Concurrency** | Concurrent reads | Concurrent with other shared |
| **Blocking** | May wait for cache | No blocking |
| **History impact** | None | None |

---

## 12. Versioning & Deployments

### Temporal Versioning

```kotlin
// TEMPORAL: Workflow.getVersion for safe deployments

override fun process(payments: List<String>): Result {
    // Version check for code changes
    val version = Workflow.getVersion(
        "batch-size-change",           // Change ID
        Workflow.DEFAULT_VERSION,      // Min version (old code)
        1                              // Max version (new code)
    )

    val batchSize = if (version == Workflow.DEFAULT_VERSION) {
        10  // Old behavior
    } else {
        5   // New behavior (version 1)
    }

    // Use batchSize...
}

// Versioning rules:
// - Running workflows get DEFAULT_VERSION from history
// - New workflows get max version (1)
// - Both code paths must exist during migration
// - Remove old path only after all old workflows complete
```

### Restate Versioning

```typescript
// RESTATE: Different approach - handler replacement

// Restate's journal-based replay is more forgiving:
// - Only journaled operations need to match
// - Code changes between ctx.* calls are safe
// - State schema changes need migration

// Safe changes (no special handling):
// - Adding new ctx.run() calls at the end
// - Changing non-journaled code
// - Adding new handlers

// Requires care:
// - Changing order of ctx.run() calls
// - Removing ctx.run() calls
// - Changing ctx.run() return types

// Migration pattern for breaking changes:
const paymentService = restate.service({
  name: "payment",
  handlers: {
    // Keep old handler
    processV1: async (ctx: Context, payments: string[]) => {
      // Old implementation
    },

    // Add new handler
    processV2: async (ctx: Context, payments: string[]) => {
      // New implementation
    },

    // Router handler (optional)
    process: async (ctx: Context, req: { version: number, payments: string[] }) => {
      if (req.version === 1) {
        return ctx.serviceClient(paymentService).processV1(req.payments);
      }
      return ctx.serviceClient(paymentService).processV2(req.payments);
    }
  }
});
```

### Comparison

| Aspect | Temporal | Restate |
|--------|----------|---------|
| **Versioning mechanism** | `Workflow.getVersion()` | Handler versioning / careful changes |
| **Code path branching** | Explicit in workflow | Separate handlers |
| **Replay sensitivity** | Very strict | More forgiving |
| **Migration complexity** | High (determinism) | Medium (journal matching) |
| **Rollback** | Difficult | Handler routing |

---

## 13. Code Pattern Comparisons

### Pattern 1: Parallel Activity Execution

```kotlin
// TEMPORAL
val promises = paymentIds.map { id ->
    Async.function { activities.processPayment(id) }
}
val results = promises.map { it.get() }
```

```typescript
// RESTATE
const results = await Promise.all(
  paymentIds.map(id =>
    ctx.run(`process-${id}`, () => processPayment(id))
  )
);
```

### Pattern 2: Sequential with State Updates

```kotlin
// TEMPORAL
paymentIds.forEach { id ->
    val result = activities.processPayment(id)
    processedCount++  // State via replay
    results.add(result)
}
```

```typescript
// RESTATE (Virtual Object)
for (const id of paymentIds) {
  const result = await ctx.run(`process-${id}`, () => processPayment(id));
  const count = (await ctx.get<number>("processedCount") ?? 0) + 1;
  ctx.set("processedCount", count);

  const results = await ctx.get<string[]>("results") ?? [];
  results.push(result);
  ctx.set("results", results);
}
```

### Pattern 3: Wait for External Event

```kotlin
// TEMPORAL
@SignalMethod
override fun onApproval(approved: Boolean) {
    this.approved = approved
}

override fun waitForApproval(): Boolean {
    Workflow.await { approved != null }
    return approved!!
}
```

```typescript
// RESTATE (Awakeable)
const awakeable = ctx.awakeable<boolean>();
// Share awakeable.id with external system
const approved = await awakeable.promise;
return approved;
```

### Pattern 4: Retry with Custom Logic

```kotlin
// TEMPORAL
var attempts = 0
while (attempts < 3) {
    try {
        return activities.callExternalService()
    } catch (e: ActivityFailure) {
        attempts++
        if (attempts >= 3) throw e
        Workflow.sleep(Duration.ofSeconds(attempts * 10))
    }
}
```

```typescript
// RESTATE
let attempts = 0;
while (attempts < 3) {
  try {
    return await ctx.run("call-service", () => callExternalService());
  } catch (e) {
    if (e instanceof restate.TerminalError) throw e;
    attempts++;
    if (attempts >= 3) throw new restate.TerminalError("Max retries exceeded");
    await ctx.sleep(attempts * 10000);
  }
}
```

### Pattern 5: Fan-out / Fan-in (Child Workflows)

```kotlin
// TEMPORAL
val childPromises = gateways.map { gateway ->
    val child = Workflow.newChildWorkflowStub(GatewayWorkflow::class.java, options)
    Async.function { child.process(gateway, payments) }
}
val results = childPromises.map { it.get() }
return aggregate(results)
```

```typescript
// RESTATE
const results = await Promise.all(
  gateways.map(gateway =>
    ctx.objectClient(gatewayProcessor, gateway).process(payments)
  )
);
return aggregate(results);
```

---

## 14. Key Questions for AI Agent

Use these questions when asking an AI agent to explain concepts:

### Fundamental Concepts

1. **"How does Restate's journal-based replay differ from Temporal's event sourcing? What are the practical implications?"**

2. **"In Temporal, workers poll for tasks. In Restate, the server pushes to services. What are the trade-offs of each approach?"**

3. **"Explain Restate's Virtual Objects vs Temporal Workflows. When would I use each?"**

4. **"How do I achieve the equivalent of Temporal's @SignalMethod in Restate?"**

5. **"What code changes are safe in Restate vs Temporal without breaking running executions?"**

### Migration Questions

6. **"I have a Temporal workflow that uses child workflows for parallel processing. How should I structure this in Restate?"**

7. **"My Temporal activities have specific retry policies. How do I configure equivalent behavior in Restate?"**

8. **"In Temporal, I use Workflow.await() with a timeout. What's the Restate equivalent?"**

9. **"How do I migrate Temporal's query methods to Restate?"**

10. **"My Temporal workflow uses Workflow.getVersion() for deployments. What's the Restate approach?"**

### Advanced Patterns

11. **"How does Restate handle the equivalent of Temporal's sticky execution and workflow caching?"**

12. **"In Temporal, activity results are stored in history and replayed. How does Restate store and replay side effect results?"**

13. **"Explain Restate's awakeables. How do they compare to Temporal's signals?"**

14. **"How would I implement the saga pattern in Restate vs Temporal?"**

15. **"What happens in Restate when a service handler crashes mid-execution? How does recovery work compared to Temporal?"**

### Architecture Questions

16. **"Can I have multiple Restate services call each other like Temporal workers with different task queues?"**

17. **"How does Restate's state storage compare to Temporal's history storage in terms of limits and performance?"**

18. **"In Temporal, I can query workflow history for debugging. What's the equivalent in Restate?"**

19. **"How do I implement distributed transactions/sagas that span multiple services in Restate?"**

20. **"What observability tools does Restate provide compared to Temporal's Web UI and tctl?"**

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TEMPORAL → RESTATE QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TEMPORAL                          RESTATE                                  │
│  ─────────────────────────────────────────────────────────────────────────  │
│  @WorkflowMethod                   Service handler / Object handler         │
│  @ActivityMethod                   ctx.run(() => ...)                       │
│  @SignalMethod                     Object handler + state                   │
│  @QueryMethod                      Shared handler                           │
│                                                                             │
│  Workflow.sleep()                  ctx.sleep()                              │
│  Workflow.await { }                Awakeable + Promise.race                 │
│  Workflow.getVersion()             Handler versioning                       │
│  Workflow.sideEffect()             ctx.run() or direct code                 │
│                                                                             │
│  Async.function { }                Promise (with ctx.run inside)            │
│  promise.get()                     await promise                            │
│  Promise.allOf()                   Promise.all()                            │
│                                                                             │
│  Child Workflow                    ctx.serviceClient() / ctx.objectClient() │
│  WorkflowClient.start()            HTTP call to Restate                     │
│  workflowStub.signal()             Object handler call                      │
│  workflowStub.query()              Shared handler call                      │
│                                                                             │
│  RetryOptions                      ctx.run options / global config          │
│  ActivityOptions                   ctx.run options                          │
│                                                                             │
│  Workflow.currentTimeMillis()      Date.now() (outside ctx.run)             │
│  Workflow.randomUUID()             crypto.randomUUID() (outside ctx.run)    │
│                                                                             │
│  WorkflowExecutionTimeout          Invocation timeout config                │
│  ActivityStartToCloseTimeout       ctx.run timeout option                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```