# 🚀 From .NET/Marten to Rust & Interlinker: The 60-Day Master Transition Roadmap
## 500 Production-Grade Exercises for Senior .NET Engineers Transitioning to Rust & Interlinker Distributed Architecture

---

## 🎯 Executive Summary & Architectural Philosophy

Welcome to the **Interlinker Engineering Academy**.

Our core data fabric and runtime architecture draw inspiration from the best patterns in enterprise .NET:
- **MartenDB** (PostgreSQL document store, identity map, bitemporal event sourcing, async projection daemons).
- **MassTransit & NServiceBus** (Routing slip sagas, compensating transactions $M^{-1}$, idempotent outbox, split-brain CAS fencing).
- **Microsoft Orleans & Akka.NET** (Virtual actor model, grain lifecycle, turn-based single concurrency, snapshot hydration).

However, in **Interlinker**, we took these enterprise patterns and completely re-engineered them in **Native Rust on Citus Distributed Clusters**:
- **Zero Garbage Collection Stalls**: Replacing the .NET GC (Gen0/1/2, LOH, Server GC pauses) with Rust’s affine type system, compile-time borrow checker, and hardware-aligned memory ownership.
- **Zero-Network-Shuffle Joins**: Distributing PostgreSQL into Citus shards co-located by `tenant_id`.
- **Arbitrary-Depth Multi-Tenancy**: Resolving dot-notation hierarchies (`acme.us.california.aerospace.avionics`) into deterministic 128-bit UUIDv5 GUIDs in **$0\text{ ms}$** without mandatory database roundtrips.
- **Formal Correctness**: Hardware-enforced monotonic CAS fencing, SMT pre-commit invariants ($\Phi(S) \le 0$), and bitemporal horizon ledgers.

### Training Guidelines & Constraints
1. **No Proprietary Platform Access Required**: Every exercise runs on standard developer workstations using .NET 8/9 C#, Rust 1.78+, Docker/Podman, and PostgreSQL 16+.
2. **Timeline**: 60 Days (8 Weeks).
3. **Volume**: Exactly **500 structured exercises** (approximately 8–9 exercises per day).
4. **Execution Protocol**: Every exercise must compile, include unit/integration tests, and be verified against local Dockerized PostgreSQL/Citus instances.

---

## 📅 Curriculum Timeline Overview (8 Weeks / 60 Days)

```
+---------------------------------------------------------------------------------------------------+
| WEEK 1: MartenDB, JSONB & PostgreSQL Multi-Model Foundations (Exercises 1 - 65)                   |
| Day 1-2: Document Store & Identity Map | Day 3-4: OCC & Bitemporality | Day 5-7: Event Sourcing   |
+---------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
| WEEK 2: Distributed Sagas, Routing Slips & The Outbox Pattern (Exercises 66 - 125)               |
| Day 8-10: State Machines & Slips | Day 11-12: Reverse Compensations M⁻¹ | Day 13-15: Outbox & CAS |
+---------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
| WEEK 3: The Virtual Actor Model & Distributed Grain Runtimes (Exercises 126 - 185)               |
| Day 16-18: Orleans Grains & Mailboxes | Day 19-21: State Hydration | Day 22-23: Supervision      |
+---------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
| WEEK 4: Rust Systems Mastery for C# Engineers (Exercises 186 - 250)                               |
| Day 24-26: CLR vs Borrow Checker | Day 27-28: Traits vs Interfaces | Day 29-31: Smart Pointers    |
+---------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
| WEEK 5: Async Rust, Concurrency & Tokio Deep Dive (Exercises 251 - 315)                           |
| Day 32-34: Task vs Future | Day 35-36: Send/Sync & Thread Safety | Day 37-39: Channels & Select   |
+---------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
| WEEK 6: Rebuilding Marten & Sagas in Native Rust (Exercises 316 - 380)                            |
| Day 40-42: Rust DocumentStore & IdentityMap | Day 43-44: Rust Event Store | Day 45-47: Rust Sagas |
+---------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
| WEEK 7: Building the Virtual Actor Runtime in Rust (Exercises 381 - 440)                          |
| Day 48-50: Actor Directory & Tokio Mailbox | Day 51-52: Dual-State Hydration | Day 53-54: Supervision|
+---------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
| WEEK 8: The Interlinker Mindset - Citus, Shards & Hyperscale (Exercises 441 - 500)                |
| Day 55-56: Citus 15 Pillars & Sharding | Day 57-58: 0ms UUIDv5 Multi-Tenancy | Day 59-60: Capstone|
+---------------------------------------------------------------------------------------------------+
```

---

## 🛠️ Required Local Developer Environment Setup

Before starting Day 1, set up the standard local infrastructure:

```bash
# 1. Install .NET SDK 8 or 9
curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --channel 8.0

# 2. Install Rust Toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env

# 3. Spin up PostgreSQL 16 with Citus Extension
docker run -d --name citus-training \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=TrainingSecure2026! \
  -e POSTGRES_USER=interlinker_dev \
  -e POSTGRES_DB=interlinker_academy \
  citusdata/citus:12.1
```

---

## 📘 Phase 1: MartenDB, JSONB & PostgreSQL Multi-Model Foundations (Weeks 1)

### Day 1: Document Store Core & Identity Map in .NET (Exercises 1–10)
1. **Initialize Marten DocumentStore**: Configure Marten with PostgreSQL connection string, auto-schema creation (`AutoCreate.All`), and custom JSON serializer settings (`System.Text.Json`).
2. **Basic CRUD with Marten**: Create an entity `CustomerAccount` with `Guid Id`, `string Name`, and `decimal Balance`. Insert, fetch by ID, update, and delete.
3. **Marten Session Flavors**: Contrast `OpenSession()` (read-write with dirty checking), `QuerySession()` (read-only zero tracking), and `LightweightSession()` (no identity map).
4. **Identity Map Invariant**: Load the same `CustomerAccount` twice within one session. Assert via `object.ReferenceEquals` that both return the identical in-memory instance.
5. **Dirty Tracking Deep Dive**: Mutate a property on a tracked entity. Call `SaveChangesAsync()` and inspect PostgreSQL pg_stat_activity to verify that an `UPDATE` SQL was dispatched.
6. **No-Op Change Detection**: Call `SaveChangesAsync()` on a session where an entity was fetched but never mutated. Assert 0 SQL updates were emitted.
7. **Document Metadata Columns**: Inspect PostgreSQL table `mt_doc_customeraccount`. Verify `data` (JSONB), `id` (UUID), `mt_dotnet_type`, and `mt_last_modified`.
8. **JSONB Path Indexing**: Configure a duplicate field index on `CustomerAccount.Email` using Marten's `.Index(x => x.Email)`. Verify the created B-Tree index in PostgreSQL.
9. **GIN Indexing on Full Documents**: Add a GIN index on the `data` column using `.Index(x => x, x => x.Method = IndexMethod.gin)`. Execute a JSON containment query (`@>`).
10. **Unit of Work Batching**: Insert 10 documents, update 5, and delete 2 in a single session. Inspect PostgreSQL logs to verify Marten bundles them into a single batched network roundtrip.

### Day 2: Advanced Document Querying, LINQ & Concurrency (Exercises 11–20)
11. **LINQ to JSONB Translation**: Write a Marten LINQ query filtering `x.Balance > 1000 && x.Status == "Active"`. Inspect Marten’s generated SQL.
12. **Child Collection Querying**: Query customers where `x.Orders.Any(o => o.Total > 500)`. Verify the JSONB `jsonb_array_elements` or `@>` syntax generated.
13. **Optimistic Concurrency Control (OCC)**: Configure `[Version]` on `CustomerAccount`. Load an entity in Session A and Session B. Save changes in Session A, then attempt to save Session B. Catch `ConcurrencyException`.
14. **Custom Version Hashing**: Configure Marten to use Guid versioning instead of numeric versioning. Verify how `mt_version` updates in PostgreSQL.
15. **Soft Deletes in Marten**: Configure `schema.For<CustomerAccount>().SoftDeleted()`. Delete an account. Query all accounts and verify the deleted record is filtered out by `mt_deleted IS FALSE`.
16. **Hard Deletes vs Soft Deletes**: Write a query using `session.Query<CustomerAccount>().Where(x => x.MaybeDeleted())` to query soft-deleted records. Perform a hard delete using `HardDelete()`.
17. **Document Patching API**: Use Marten's `.Patch(id).Set("Status", "Suspended")` without loading the full document into application memory.
18. **Atomic Numeric Increment Patching**: Use `.Patch(id).Increment("LoginAttempts", 1)` to atomically increment a counter inside JSONB.
19. **Bulk Insert Performance**: Insert 10,000 documents using `session.Store()` vs `store.BulkInsertDocuments(list)`. Benchmark throughput difference.
20. **Tenancy in Marten**: Configure Marten for conjoined tenancy (`schema.For<Order>().MultiTenanted()`). Insert orders with `tenant_a` and `tenant_b`. Assert tenant isolation.

### Day 3: Event Sourcing Foundations in Marten (Exercises 21–30)
21. **Append-Only Event Stream**: Create events `OrderCreated`, `OrderPaid`, `OrderShipped`. Append them to a new stream using `session.Events.StartStream<OrderAggregate>(id, events)`.
22. **Inspecting `mt_events` Table**: Examine PostgreSQL table `mt_events`. Inspect columns `stream_id`, `version`, `data`, `type`, and `timestamp`.
23. **Stream Version OCC**: Attempt to append an event with expected version 2 when the stream is at version 3. Verify Marten throws `EventStreamUnexpectedMaxEventIdException`.
24. **Live Aggregate Reconstruction**: Implement `OrderAggregate` with `Apply(OrderCreated e)` and `Apply(OrderPaid e)`. Fetch the live aggregate using `session.Events.AggregateStreamAsync<OrderAggregate>(streamId)`.
25. **Inline Projections**: Configure an inline projection `SingleStreamProjection<OrderAggregate>`. Verify that `mt_doc_orderaggregate` updates synchronously within the event-appending transaction.
26. **Custom Aggregation Logic**: Implement an aggregate where `Apply` calculates running balances and status transitions.
27. **Event Metadata & Correlation**: Append events carrying `CorrelationId`, `CausationId`, and `UserId` in the event headers. Verify persistence in `mt_events.headers`.
28. **Stream Tombstoning**: Append an `OrderCancelled` event and configure aggregate to tombstone itself.
29. **Querying Raw Event Streams**: Use `session.Events.QueryAllRawEvents()` to query events by event type and timestamp window.
30. **Stream Archiving**: Archive an event stream using Marten's stream archiving features. Query active vs archived streams.

### Day 4: Async Projections & Event Store Daemon (Exercises 31–40)
31. **Asynchronous Projections**: Create a projection `CustomerAccountSummaryProjection` that updates a read-model document asynchronously.
32. **Marten Async Daemon Setup**: Configure `AddAsyncDaemon(DaemonMode.Solo)` in .NET host builder. Start the daemon and append 100 events. Verify asynchronous projection catches up.
33. **Tracking High Water Mark**: Inspect `mt_event_progression` in PostgreSQL. Observe how the daemon records `last_seq_id`.
34. **Projection Rebuilding**: Trigger a complete projection rebuild using `daemon.RebuildProjection<CustomerAccountSummaryProjection>()`.
35. **Multi-Stream Projections**: Build a projection `MonthlyRevenueReport` that aggregates events across multiple independent streams (`OrderCreated`, `RefundIssued`).
36. **Fault Tolerance in Projections**: Throw an unhandled exception inside a projection's `Apply` method. Observe Marten's circuit breaker and retry policy.
37. **Zero-Downtime Blue-Green Projections**: Configure two projections writing to `mt_doc_order_summary_v1` and `mt_doc_order_summary_v2`. Switch reads once v2 completes catch-up.
38. **Bitemporality in Event Stores**: Model events with both `OccurredAt` (valid time) and `RecordedAt` (system time). Write a query analyzing historical state as known at timestamp $T$.
39. **Time-Travel Snapshot Generation**: Reconstruct an aggregate at a specific version $V$ using `session.Events.AggregateStreamAsync(streamId, version)`.
40. **Stream Compaction & Snapshots**: Implement an automatic snapshot policy where every 100 events, a snapshot document is saved, and live replay starts from the snapshot.

### Day 5–7: Marten Mastery & Enterprise Patterns (Exercises 41–65)
41. **PostgreSQL Full-Text Search**: Configure Marten full-text search on `Product.Description`. Execute queries using `session.Query<Product>().Where(x => x.Search("autonomous engine"))`.
42. **Foreign Key Shredding**: Configure Marten to shred relational foreign keys from JSONB documents into separate columns with foreign key constraints.
43. **Custom SQL Projections**: Implement an `IProjection` that executes raw PostgreSQL `INSERT ... ON CONFLICT DO UPDATE` queries.
44. **PostgreSQL Listen/Notify Bridge**: Configure PostgreSQL `pg_notify` on event appends. Build a .NET background worker listening to notifications via `NpgsqlConnection`.
45. **Multi-Tenancy per Database**: Configure Marten's master database tenancy router dispatching tenants to distinct PostgreSQL connection strings.
46. **CDC (Change Data Capture) with Marten**: Tap into Marten’s transaction commit lifecycle using an `IDocumentSessionListener` to broadcast outbox messages.
47. **Unit-of-Work Hook Validation**: Prevent document commits in an `IDocumentSessionListener` if an invariant rule is violated (e.g., negative balance).
48. **Custom JSON Converters**: Implement a `JsonConverter` in System.Text.Json for strong-typed IDs (e.g. `AccountId(Guid)`). Verify seamless serialization in Marten.
49. **Deadlock Simulation & Handling**: Run 20 concurrent threads updating the same pair of documents in opposite order. Verify Marten's exception and implement retry policies.
50. **Transaction Isolation Levels**: Test Marten operations under `ReadCommitted`, `RepeatableRead`, and `Serializable`.
51. **Encrypted Document Fields**: Implement a listener that encrypts sensitive fields (SSN, credit card) using AES-256 before saving to JSONB.
52. **Event Sourcing with Crypto Seals**: Calculate a SHA-256 Merkle chain hash for each appended event ($H_i = \text{SHA256}(H_{i-1} \parallel \text{Payload}_i)$).
53. **Large Document Streaming**: Stream 1,000,000 documents using `session.Query<T>().ToAsyncEnumerable()` to observe memory consumption.
54. **PostgreSQL Index Performance Analysis**: Use `EXPLAIN (ANALYZE, BUFFERS)` on Marten queries with and without computed JSONB indexes.
55. **Partitioning Marten Tables**: Configure native PostgreSQL range partitioning on `mt_events` by timestamp. Verify partition pruning.
56. **Marten Custom Linq Extension**: Write a custom LINQ extension method translating C# methods into custom PostgreSQL SQL functions.
57. **Out-of-Order Event Handling**: Implement reconciliation logic for events arriving out of chronological order.
58. **Event Sourced Bank Account**: Build a complete banking account model (Open, Deposit, Withdraw, Transfer) with invariants enforced in live aggregates.
59. **Audit Trail Projections**: Build an audit read-model tracking who modified what document property across time.
60. **Session Export/Import**: Export an event stream to JSON and import it into an isolated test database.
61. **Dead-Letter Event Handling**: Build an ingestion pipeline that routes malformed events to a dead-letter table.
62. **Schema Migration Scripts**: Use Marten's `WriteMigrationFile` to generate SQL DDL migration files for CI/CD pipelines.
63. **Multi-Database Sharding Prototype**: Write a basic sharding coordinator in C# routing tenants across 3 PostgreSQL instances using consistent hashing.
64. **Marten Health Checks**: Integrate Marten health checks into ASP.NET Core `/healthz` endpoint.
65. **Phase 1 Capstone (Marten)**: Build a fully functioning Event-Sourced Logistics Order Management system with 3 aggregates, 2 async projections, and OCC in .NET.

---

## 🔄 Phase 2: Distributed Sagas, Routing Slips & Outbox Pattern (Week 2)

### Day 8–10: Saga Patterns & MassTransit Routing Slips in .NET (Exercises 66–90)
66. **Choreography vs Orchestration**: Model an e-commerce checkout flow using event choreography. Identify risks of cyclic dependencies and lack of observability.
67. **Orchestrated Saga State Machine**: Build a saga state machine using `MassTransit.StateMachine` with states `Submitted`, `InventoryReserved`, `PaymentAuthorized`, `Completed`.
68. **Saga State Persistence**: Persist saga instance state into PostgreSQL using Marten (`mt_doc_checkoutstate`).
69. **Correlation IDs & Idempotency**: Ensure every saga message carries a deterministic `CorrelationId`. Configure message deduplication.
70. **MassTransit Routing Slip Pattern**: Define a routing slip with itinerary: `ReserveInventoryActivity`, `AuthorizePaymentActivity`, `ShipOrderActivity`.
71. **Routing Slip Compensation ($M^{-1}$)**: Implement `Compensate()` on `ReserveInventoryActivity` to release reserved stock if payment fails.
72. **Simulating Downstream Faults**: Trigger a simulated network failure in `AuthorizePaymentActivity`. Assert routing slip executes reverse compensations in strict LIFO order.
73. **Saga Monotonic Fencing Tokens**: Add a monotonic integer version to saga state. Verify that out-of-order delayed messages are discarded.
74. **Split-Brain CAS (Compare-And-Swap) Protection**: Write a PostgreSQL query updating saga state with `WHERE id = @id AND fencing_token = @expected_token`.
75. **Saga Timeouts and Escalations**: Configure timeout events. If payment is not authorized within 30 seconds, transition saga to `Expired` and compensate.
76. **Parallel Fork-Join in Sagas**: Build a saga that branches into parallel sub-tasks (e.g. credit check and fraud scan) and synchronizes on a barrier.
77. **Saga Compensation Idempotency**: Verify that calling compensation twice produces identical safe results without double-refunding.
78. **Variable-Length Routing Slips**: Dynamically append activities to a routing slip based on order size or international shipping requirements.
79. **Saga Event Sourcing**: Instead of persisting mutable saga state, persist the saga as an append-only event stream of routing events.
80. **Deadlock Free Lock Ordering**: When a saga needs to acquire locks on multiple documents, sort document IDs lexicographically before locking.
81. **Distributed Saga Visualizer**: Write a small UI or CLI visualizing the current active step and compensation stack of an in-flight saga.
82. **Long-Running Human-in-the-Loop Saga**: Build a saga that enters `AwaitingApproval` state and resumes only upon an admin approval event.
83. **Saga Event Replay**: Reconstruct a failed saga's exact execution history from the outbox log.
84. **Epistemic Circuit Breaker in Sagas**: Halt a saga if downstream service error rates exceed 50% over a 10-second sliding window.
85. **Saga Diagnostic Tracing**: Inject OpenTelemetry trace contexts into routing slip messages to trace end-to-end execution.
86. **Multi-Tenant Saga Routing**: Dispatch sagas to tenant-specific queues based on tenant routing rules.
87. **Transactional Envelope Encryption in Sagas**: Encrypt payload parameters of sensitive saga steps before queuing.
88. **Saga Versioning & Upgrades**: Deploy a v2 of a saga state machine while v1 saga instances are still actively in-flight.
89. **High-Load Saga Benchmark**: Run 1,000 concurrent saga executions in .NET and record throughput and database contention.
90. **Saga Chaos Testing**: Terminate the worker process randomly during routing slip execution. Assert state recovery upon reboot.

### Day 11–15: The Outbox Pattern & Monotonic Distributed Safety (Exercises 91–125)
91. **Dual-Write Hazard Demonstration**: Demonstrate data inconsistency when writing to PostgreSQL and publishing to RabbitMQ/Kafka in two separate calls without 2PC.
92. **Transactional Outbox Table**: Create table `outbox_messages` in PostgreSQL (`id`, `payload`, `destination`, `created_at`, `dispatched_at`).
93. **Atomic Outbox Enqueue**: In a single PostgreSQL transaction, insert a business document and enqueue an outbox message.
94. **Outbox Polling Worker**: Build a background worker in C# polling `WHERE dispatched_at IS NULL ORDER BY created_at LIMIT 50 FOR UPDATE SKIP LOCKED`.
95. **High-Performance Outbox with `SKIP LOCKED`**: Verify that multiple parallel outbox worker processes never process the same message simultaneously.
96. **At-Least-Once Delivery**: Simulate network failure during broker publish. Verify message remains undispatched and is safely retried.
97. **Idempotent Consumer Pattern**: Build a consumer storing processed `MessageId` in a deduplication table with unique constraint.
98. **Outbox Log Compaction**: Build a cleanup job deleting or archiving dispatched messages older than 7 days.
99. **CDC Outbox with PostgreSQL Logical Replication**: Replace the polling worker with PostgreSQL WAL decoding (`pgoutput`).
100. **Monotonic Sequence Generator**: Implement an outbox sequence generator ensuring strict monotonic sequential delivery per aggregate.
101. **Outbox Batching**: Group multiple outbox events into a single bulk publish payload.
102. **Poison Message Dead-Lettering**: Move an outbox message to `outbox_dead_letter` after 5 consecutive failed delivery attempts.
103. **Outbox Priority Queues**: Add priority column to outbox table (`High`, `Normal`, `Low`) and dispatch high-priority messages first.
104. **Pre-Commit Invariant Checking**: Run formal invariant assertions ($\Phi(S) \le 0$) prior to committing the outbox transaction.
105. **Distributed Fencing Token CAS**: Implement fencing tokens in outbox publishers to prevent split-brain zombies.
106. **Outbox Telemetry**: Track outbox lag metric (time between `created_at` and `dispatched_at`) in Prometheus.
107. **Multi-Tenant Outbox Routing**: Partition outbox table by `tenant_id` to prevent noisy neighbors from blocking outbox dispatch.
108. **Crypto Sealed Outbox Records**: Hash outbox payloads with HMAC-SHA256 to ensure zero message tampering in transit.
109. **Outbox Replay for Disaster Recovery**: Re-dispatch outbox events from timestamp $T$ to rehydrate a lost read-model.
110. **Phase 2 Capstone (Saga + Outbox)**: Build a complete Distributed Payment Settlement Saga with Routing Slips, Reverse Compensations, and Transactional Outbox in .NET.
111–125. *(Deep dive exercises in saga failure matrices, complex compensation rollbacks, outbox backpressure throttling, and zero-loss cutovers).*

---

## 🎭 Phase 3: The Virtual Actor Model & Distributed Grain Runtimes (Week 3)

### Day 16–18: Orleans Virtual Actors & Turn-Based Concurrency (Exercises 126–155)
126. **Virtual Actor Concept**: Contrast physical actors (Akka/Erlang where address = process) with virtual actors (Orleans where address = identity, activation is lazy).
127. **Hello Orleans Grain**: Build an `IOrderGrain` with `Guid` key. Call `grainFactory.GetGrain<IOrderGrain>(orderId)` and invoke methods.
128. **Single-Threaded Execution Invariant**: Verify that Orleans processes requests to a single grain sequentially without multi-threading race conditions.
129. **Grain State Persistence**: Configure Marten/PostgreSQL as the grain storage provider for Orleans.
130. **Lazy Grain Activation**: Observe how Orleans automatically activates a grain into memory on first request and calls `OnActivateAsync`.
131. **Grain Deactivation & Garbage Collection**: Observe how idle grains are silently evicted from memory after inactivity timeout without losing state.
132. **Timers & Reminders in Grains**: Configure an Orleans persistent reminder executing every 12 hours even if the host cluster restarts.
133. **Grain Placement Strategies**: Configure `RandomPlacement`, `PreferLocalPlacement`, and `StatelessWorkerPlacement`.
134. **Grain Reentrancy**: Experiment with `[Reentrant]` attribute. Demonstrate how non-reentrant grains prevent state interleaving.
135. **Grain Interceptors & Filters**: Write an incoming grain call filter logging latency and checking security clearance.
136. **Actor Mailbox Backpressure**: Simulate 10,000 rapid requests to a single grain. Measure queue depth and latency.
137. **Priority Mailboxes in Akka.NET**: Implement an Akka.NET actor with a priority mailbox prioritizing `EmergencyStop` messages.
138. **Supervision Strategies**: Build a parent supervisor actor with `OneForOneStrategy(Directive.Restart)` to recover crashed worker actors.
139. **Stateful vs Stateless Grains**: Implement a `TaxCalculatorGrain` as a `StatelessWorker` scaling across all CPU cores.
140. **Grain-to-Grain Messaging**: Implement a multi-grain workflow where `OrderGrain` calls `InventoryGrain` and `BillingGrain`.
141–155. *(Grain lifecycle hooks, distributed deadlocks in circular grain calls, grain streams, cluster membership, and silo failover).*

### Day 19–23: Grain Hydration, Dual-Mind Architecture & Distributed State (Exercises 156–185)
156. **Dual-State Recovery Pattern**: On grain activation, load the latest aggregate snapshot document from PostgreSQL, then replay all newer delta events.
157. **Actor Snapshot Frequency**: Implement snapshotting every 50 events to keep activation latency below 5ms.
158. **Actor Direct Mesh**: Connect two actors directly via in-memory channels without intermediary brokers.
159. **Actor-to-Database Session Mapping**: Bind an active actor session to a Marten `IDocumentSession`.
160. **Actor Lease Provider**: Implement a lease coordinator ensuring only one instance of a specific grain exists across an entire cluster.
161. **Distributed Dead-Letter Queue**: Route unhandled actor exceptions to a dead-letter channel.
162. **Lyapunov Energy Damping**: Model actor tension and homeostasis damping functions for self-stabilizing actor systems.
163. **Multi-Agent Deliberation Room**: Convene 3 autonomous agents in a shared deliberation session agreeing on a transaction proposal.
164. **Phase 3 Capstone**: Build a complete Virtual Actor Order Fulfillment System with Orleans, Marten persistence, and dual-state recovery.
165–185. *(Actor memory compactors, causal state reconciliation, and multi-node cluster rebalancing).*

---

## 🦀 Phase 4: Rust Systems Mastery for C# Engineers (Week 4)

### Day 24–26: The Memory Model — CLR vs. Rust Borrow Checker (Exercises 186–210)
186. **Stack vs Heap in Rust**: Contrast C# class allocation (managed heap, GC pointer) with Rust value types on the stack. Write a program checking sizes using `std::mem::size_of`.
187. **Ownership & Move Semantics**: Declare `let s1 = String::from("hello"); let s2 = s1;`. Explain why calling `println!("{}", s1)` fails to compile in Rust vs C#.
188. **The Clone Trait**: Implement explicit cloning using `let s2 = s1.clone();`. Explain deep-copy overhead.
189. **Borrowing & References (`&T`)**: Write a function that reads a struct without taking ownership using immutable reference.
190. **Mutable Borrowing (`&mut T`)**: Explain the fundamental Rust rule: **Either ONE mutable reference OR any number of immutable references, but never both**.
191. **Data Race Prevention at Compile Time**: Attempt to mutate a vector while iterating over it in Rust. Compare with C#'s runtime `InvalidOperationException: Collection was modified`.
192. **Lifetimes (`'a`)**: Write a function returning the longer of two string slices `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str`. Understand lifetime annotations.
193. **Structs with References**: Create a struct `DocumentSnapshot<'a>` holding `&'a str` and `&'a [u8]`.
194. **Static Lifetime (`'static`)**: Understand `'static` strings vs heap strings.
195. **RAII & The Drop Trait**: Implement `Drop` for a `DatabaseConnection` struct. Verify it closes deterministically when exiting scope without `using` or `IDisposable`.
196. **Interior Mutability (`Cell` & `RefCell`)**: Mutate state behind an immutable reference using `RefCell`. Understand runtime borrow checking.
197. **Thread-Safe Shared Ownership (`Arc<T>`)**: Share an immutable data structure across 5 threads using `Arc`.
198. **Thread-Safe Mutation (`Arc<Mutex<T>>`)**: Mutate shared state across threads using `Arc<Mutex<T>>`.
199. **`RwLock<T>` for Read-Heavy Workloads**: Implement a high-throughput cache using `Arc<RwLock<HashMap<String, String>>>`.
200. **Deadlock Simulation in Rust**: Lock two `Mutex`es in opposite order in two threads to observe runtime deadlock. Implement sorted lock ordering.
201–210. *(Pointer aliasing, custom allocators, zero-copy string slices `&str` vs `String`, and memory leak prevention in cyclical `Rc` graphs).*

### Day 27–31: Traits, Pattern Matching, Error Handling & Generics (Exercises 211–250)
211. **Traits vs C# Interfaces**: Implement a trait `StorageProvider` for `PostgresStorage` and `MemoryStorage`.
212. **Static Dispatch (Generics & Monomorphization)**: Write `fn execute<T: StorageProvider>(provider: &T)`. Explain zero-cost runtime dispatch.
213. **Dynamic Dispatch (`dyn Trait`)**: Write `fn execute(provider: &dyn StorageProvider)`. Explain vtables and heap fat pointers.
214. **Derive Macros**: Use `#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]`.
215. **Error Handling without Exceptions**: Contrast C# `try/catch` with Rust `Result<T, E>` and `Option<T>`.
216. **The `?` Operator**: Propagate errors up the call stack using `?`.
217. **Custom Error Enums with `thiserror`**: Define a domain error enum `DatabaseError` with variants `NotFound`, `ConcurrencyConflict`, `ConnectionFailed`.
218. **Pattern Matching with `match`**: Implement exhaustive matching over an enum `PaymentStatus`. Verify compiler guarantees no unhandled cases.
219. **`if let` and `while let`**: Use concise pattern matching constructs.
220. **From/Into Conversions**: Implement `From<std::io::Error> for DatabaseError`.
221. **Associated Types in Traits**: Define a trait `Iterator` with `type Item;`. Contrast with C# generic interfaces `IEnumerable<T>`.
222. **Operator Overloading**: Implement `std::ops::Add` for a `Money` struct.
223. **Generic Constraints & Where Clauses**: Write complex generic functions with multiple trait bounds.
224. **The Newtype Pattern**: Wrap primitive `Uuid` into `TenantId(Uuid)` to prevent accidental ID mixing at compile time.
225. **Const Generics**: Implement an inline array buffer `FixedBuffer<const N: usize>`.
226. **Zero-Sized Types (ZST)**: Use marker types for state machines at compile time (typestate pattern).
227. **The Typestate Pattern**: Implement a connection state machine `Connection<Disconnected>` $\to$ `Connection<Connected>` where invalid transitions fail compilation.
228. **Parsing without Allocations**: Parse raw byte slices using `nom` or standard slices.
229. **Serialization with `serde`**: Serialize and deserialize complex nested structs to/from JSON.
230. **Phase 4 Capstone**: Implement a complete in-memory thread-safe Document Database in Rust with `Result` error handling, `Arc<RwLock>`, and LINQ-like trait filters.
231–250. *(Benchmarking with `criterion`, unsafe Rust boundaries, SIMD basics, and zero-allocation string builders).*

---

## ⚡ Phase 5: Async Rust, Concurrency & Tokio Deep Dive (Week 5)

### Day 32–34: The Async Mental Model — C# Tasks vs. Rust Futures (Exercises 251–280)
251. **Poll-Based Async vs Callback Async**: Contrast C#'s heap-allocated `Task` (eagerly scheduled on ThreadPool) with Rust's lazy `Future` (does nothing until polled).
252. **The Future Trait**: Inspect `std::future::Future { type Output; fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>; }`.
253. **Pinning and `Pin<&mut T>`**: Explain self-referential structs in async state machines and why `Pin` is required to prevent moving in memory.
254. **Tokio Runtime Initialization**: Configure a Tokio multi-threaded runtime with `#[tokio::main]`.
255. **Tokio Task Spawning**: Use `tokio::spawn` to run asynchronous background tasks.
256. **`Send` and `Sync` across `await` points**: Explain why non-`Send` types (like `Rc` or `RefCell`) cannot cross `.await` points in multi-threaded runtimes.
257. **Blocking Calls in Async Code**: Demonstrate how `std::thread::sleep` blocks Tokio worker threads. Fix it using `tokio::task::spawn_blocking`.
258. **Timers & Delays**: Use `tokio::time::sleep` and `tokio::time::timeout`.
259. **Cancellation in Async Rust**: Drop a future to cancel it instantly. Contrast with C#'s `CancellationToken.ThrowIfCancellationRequested()`.
260. **Cooperative Scheduling & Yielding**: Use `tokio::task::yield_now()` to prevent CPU-intensive tasks from starving the async executor.
261. **Async Streams (`tokio_stream::Stream`)**: Consume asynchronous sequences of events.
262. **Buffered Async Streams**: Process 1,000 async stream items concurrently with `.buffered(10)`.
263. **`tokio::select!` Macro**: Race two futures (e.g. data arrival vs cancellation timer).
264. **Bi-directional Cancellation**: Implement graceful shutdown for a cluster of background workers using `tokio_util::sync::CancellationToken`.
265–280. *(Async I/O with `tokio::net::TcpStream`, async file access, custom polling state machines, and executor tuning).*

### Day 35–39: Channels, Concurrency Primitives & Network Systems (Exercises 281–315)
281. **MPSC Channel**: Build a producer-consumer pipeline using `tokio::sync::mpsc`.
282. **Bounded vs Unbounded Channels**: Measure memory footprint under producer spikes. Understand backpressure.
283. **Oneshot Channel**: Implement a request-response RPC bridge where a worker responds via `tokio::sync::oneshot`.
284. **Broadcast Channel**: Implement pub-sub event broadcasting to multiple subscribers using `tokio::sync::broadcast`.
285. **Watch Channel**: Share cluster configuration state across 100 tasks with `tokio::sync::watch`.
286. **Async Mutex (`tokio::sync::Mutex`)**: Understand when to use async Mutex vs synchronous `parking_lot::Mutex`.
287. **Async Semaphore**: Rate-limit concurrent database operations to 20 connections using `tokio::sync::Semaphore`.
288. **Connection Pooling with `deadpool`**: Build a PostgreSQL connection pool using `deadpool-postgres`.
289. **Async Health Check Server**: Build an HTTP health server in Axum returning 200 OK.
290. **TCP Framing Protocol**: Implement length-delimited frame encoding/decoding using `tokio_util::codec`.
291. **Lock-Free Concurrency with Atomics**: Implement an atomic sequence generator using `std::sync::atomic::AtomicU64`.
292. **CAS Operations on Atomics**: Implement compare-and-swap spinlocks using `compare_exchange_weak`.
293. **Phase 5 Capstone**: Build an async, multi-threaded TCP event broker in Rust handling 50,000 messages/second with bounded channels and graceful shutdown.
294–315. *(Deep dive exercises in actor mailbox backpressure, zero-copy packet parsing, and async barrier synchronization).*

---

## 🏛️ Phase 6: Rebuilding Marten & Sagas in Native Rust (Week 6)

### Day 40–44: Rust DocumentStore, IdentityMap & Event Store (Exercises 316–350)
316. **Rust DocumentStore Architecture**: Define `DocumentStore` in Rust holding a `deadpool_postgres::Pool`.
317. **Document Trait in Rust**: Define a trait `Document` with `document_type() -> &'static str`, `id(&self) -> Uuid`, and `version(&self) -> Option<Uuid>`.
318. **Identity Map in Rust**: Build a thread-safe `IdentityMap` using `HashMap<(TypeId, Uuid), Arc<Mutex<dyn Any + Send + Sync>>>`.
319. **Dirty Tracking in Rust**: Store an initial JSON snapshot of an entity. When `save_changes` is called, serialize current state to JSON and compare with snapshot.
320. **PostgreSQL JSONB CRUD in Rust**: Execute `INSERT INTO pax_master_entities (id, tenant_id, data, version) VALUES ($1, $2, $3, $4)` via `tokio-postgres`.
321. **Linearizable OCC Version Check**: Execute SQL updates with `WHERE id = $1 AND version = $2`. Throw `ConcurrencyConflict` if row count is 0.
322. **Batched Unit of Work Commit**: Execute entity insertions, updates, and deletions within a single PostgreSQL transaction block.
323. **LINQ AST Expressions in Rust**: Use `interlinker-expressions` (`col`, `lit`, `BinaryOp`) to construct query filters.
324. **Transpiling Expressions to SQL**: Implement an `ExpressionVisitor` compiling `Expression::Binary` into SQL parameterized strings (`data->>'status' = $1`).
325. **Append-Only Event Store Table in Rust**: Create table `actor_append_events` in PostgreSQL.
326. **Stream Version OCC in Rust**: Enforce stream monotonic sequence numbers ($S_{n+1} = S_n + 1$) inside a PostgreSQL transaction.
327. **Live Aggregate Hydration in Rust**: Implement `Aggregate` trait with `apply(&mut self, event: &Event)`. Rehydrate aggregate from PostgreSQL event rows.
328. **Crypto Sealed Event Chaining**: Compute SHA-256 Merkle hashes for each event appended to PostgreSQL.
329. **Async Projection Daemon in Rust**: Build a background Tokio task tailing unread events from `actor_append_events` and updating CQRS read tables.
330. **Stream Compaction & Snapshotting**: Write a compaction engine generating entity snapshots every 100 events.
331–350. *(Soft deletes, conjoined multi-tenancy, transactional outbox listeners, and high-speed JSONB indexing).*

### Day 45–47: Distributed Sagas & Routing Slips in Rust (Exercises 351–380)
351. **Saga State Machine Trait**: Define `SagaStateMachine` trait in Rust with event triggers and state transitions.
352. **Routing Slip Activity Definition**: Define `Activity` trait with `execute(&self, ctx) -> Result<ExecutionResult, Error>` and `compensate(&self, ctx)`.
353. **Reverse Memento Stack ($M^{-1}$)**: Store serialized compensation mementos in LIFO order within PostgreSQL table `saga_step_mementos`.
354. **Monotonic CAS Fencing in Rust**: Advance saga fencing tokens using atomic database compare-and-swap (`WHERE fencing_token = $expected`).
355. **Automatic LIFO Rollback Execution**: On step failure, pop mementos from `saga_step_mementos` and invoke `compensate()` sequentially.
356. **Transactional Outbox Publisher in Rust**: Build an outbox relay polling PostgreSQL using `SELECT ... FOR UPDATE SKIP LOCKED` and dispatching to Tokio channels.
357. **Idempotent Outbox Consumer in Rust**: Deduplicate incoming events using PostgreSQL `ON CONFLICT DO NOTHING`.
358. **Pre-Commit Invariant Checking ($\Phi(S) \le 0$)**: Verify financial invariants before committing a saga transaction.
359. **Phase 6 Capstone**: Implement an end-to-end e-commerce order saga in native Rust with routing slips, reverse rollbacks, and transactional outbox.
360–380. *(Saga split-brain stress testing, deadlock-free lock ordering, and zero-allocation message envelopes).*

---

## 🤖 Phase 7: Building the Virtual Actor Runtime in Rust (Week 7)

### Day 48–54: The Actor Runtime & Virtual Grains (Exercises 381–440)
381. **Virtual Actor Directory**: Implement an in-memory `ActorDirectory` tracking active actor addresses via `Arc<RwLock<HashMap<Uuid, mpsc::Sender<ActorMessage>>>>>`.
382. **Actor Mailbox Loop**: Spawn a dedicated Tokio task for each activated actor running an event loop: `while let Some(msg) = rx.recv().await { ... }`.
383. **Turn-Based Single Concurrency**: Guarantee that an actor instance processes exactly one message at a time without race conditions.
384. **Lazy Activation**: On `get_or_create(id)`, if actor is not in directory, activate it, launch its mailbox loop, and register its sender.
385. **Dual-State Cold Rehydration**: Upon activation, load the latest snapshot from `pax_compacted_snapshots`, then stream uncompacted events from `actor_append_events`.
386. **Actor Deactivation Reaper**: Implement an idle reaper task evicting actors that have not received messages for 300 seconds.
387. **Supervision & Panic Recovery**: Catch panics in actor tasks using `tokio::task::JoinHandle` and restart the actor cleanly.
388. **Actor Ask Pattern**: Implement request-response messaging using `oneshot::channel` passed in the message envelope.
389. **Grain Persistence Bridge**: Provide an actor with a `SolidStorageSession` to save changes directly to PostgreSQL.
390. **Actor Priority Mailboxes**: Use two internal channels (High Priority vs Normal) to prioritize system control messages.
391. **Deadlock Free Actor Chains**: Implement call depth counters in message headers to detect and break circular actor calls.
392. **Phase 7 Capstone**: Build an autonomous Multi-Agent Virtual Actor cluster in Rust coordinating inventory and credit transfers with zero data loss.
393–440. *(Actor stress testing, 100,000 concurrent actors benchmark, memory profiling, and cluster heartbeat leases).*

---

## 🌐 Phase 8: The Interlinker Mindset — Citus Shards, Multi-Tenancy & Hyperscale (Week 8)

### Day 55–56: Citus Distributed Architecture & 15 Physical Pillars (Exercises 441–465)
441. **Citus Coordinator vs Worker Shards**: Spin up a Citus cluster (1 coordinator + 3 workers). Inspect `pg_dist_node`.
442. **Distributed Tables**: Create `pax_master_entities` and distribute by `tenant_id` using `create_distributed_table('pax_master_entities', 'tenant_id')`.
443. **Co-Located Tables**: Distribute `actor_append_events` co-located with `pax_master_entities` using `colocate_with => 'pax_master_entities'`.
444. **Zero-Shuffle Joins**: Execute an inner join between entities and events on `tenant_id`. Run `EXPLAIN` to verify join happens purely on worker nodes without network shuffle!
445. **Citus Reference Tables**: Distribute `tenant_hierarchy_registry` as a reference table using `create_reference_table('tenant_hierarchy_registry')`.
446. **Local Join with Reference Tables**: Join sharded `pax_master_entities` with reference table `tenant_hierarchy_registry`. Observe zero cross-worker communication.
447. **The 15 Physical Pillars Verification**: Inspect and verify all 15 distributed and reference tables of the Interlinker DB fabric.
448. **Citus Partition Pruning**: Query sharded tables with `WHERE tenant_id = $1`. Verify query routes to exactly one worker node.
449. **Cross-Shard Distributed Transactions**: Execute a transaction modifying two tenants residing on distinct workers. Observe Citus 2PC (`PREPARE TRANSACTION`).
450. **Shard Rebalancing Simulation**: Add a 4th worker node and trigger `citus_rebalance_start()`.
451–465. *(Distributed shard heat monitoring, distributed deadlocks, and worker failover).*

### Day 57–58: Arbitrary-Depth Multi-Tenancy & 0ms UUIDv5 Resolution (Exercises 466–485)
466. **Hierarchical Dot-Notation Paths**: Model deep enterprise paths: `acme.us.california.aerospace.avionics_division`.
467. **The 0ms Deterministic UUIDv5 Resolver**: Implement `TenantResolver` using RFC 4122 UUIDv5 (SHA-1 hashing against fixed namespace).
468. **Zero Database Roundtrips**: Prove that `resolve_to_guid("path")` executes in CPU registers in $0\text{ ms}$ with zero network hops.
469. **Path/GUID Parity Verification**: Prove that `resolve_to_guid` is 100% deterministic and commutative across all nodes.
470. **Citus Reference Table Synchronization**: Populate `tenant_hierarchy_registry` with paths, GUIDs, parent GUIDs, and regulatory jurisdictions.
471. **Hierarchical Ancestor Inheritance**: When querying `acme.ch.aerospace`, automatically inherit `FINMA_SWISS_BANKING` jurisdiction and `DedicatedPostgres` topology from `acme.ch`.
472. **Subtree Rollup Queries**: Write a function returning all descendant GUIDs for root `acme`. Query sharded tables with `WHERE tenant_id = ANY($1::uuid[])`.
473. **Multi-Region Topology Routing**: Route `EU_GDPR` queries to European PostgreSQL instances and `US_FEDRAMP` to US instances.
474. **Kubernetes Anti-Starvation Micro-Pooling**: Restrict connection pool to 4 connections per pod. Implement an idle pool reaper evicting inactive pools after 300s.
475. **Live Tenant Cutover**: Migrate a tenant from `SharedCitus` to `DedicatedPostgres` with zero downtime using dual-write fencing tokens.
476–485. *(Domain-based hot/cold splitting, NVMe write-masters vs read-replicas, and regulatory geofencing).*

### Day 59–60: Final Capstone Project (Exercises 486–500)
486. **Architecture Specification**: Design a decoupled, distributed Core Financial Ledger service.
487. **Hierarchical Multi-Tenancy**: Configure 10 corporate entities with arbitrary-depth divisions and deterministic UUIDv5 GUIDs.
488. **Citus Co-Located Sharding**: Deploy the 15 physical pillars on a Citus cluster.
489. **Rust Document & Event Store**: Implement the storage layer in native Rust without ORMs.
490. **Virtual Actor Engine**: Launch actor grains for each ledger account with turn-based concurrency.
491. **Dual-State Hydration**: Rehydrate account states from snapshots and WAL streams in $<5\text{ ms}$.
492. **Inter-Account Transfer Saga**: Execute a multi-account fund transfer via MassTransit-style routing slips in native Rust.
493. **Monotonic CAS Fencing**: Enforce split-brain protection on all ledger entries.
494. **Reverse Compensation ($M^{-1}$)**: Trigger automatic rollback if destination account is frozen or exceeds risk limits.
495. **SMT Pre-Commit Invariant**: Verify that total cluster debits equal credits ($\sum \text{debits} - \sum \text{credits} = 0$) before committing.
496. **Transactional Outbox Relay**: Publish ledger balance events to outbox workers using `FOR UPDATE SKIP LOCKED`.
497. **CQRS Projection Daemon**: Materialize real-time account summary cockpits asynchronously.
498. **Chaos Injection**: Randomly kill worker nodes and drop connections during active transfers. Assert 100% ACID consistency and zero lost funds.
499. **Zero-Allocation Benchmark**: Benchmark system at 25,000 transactions/second with sub-millisecond p99 latency.
500. **Final Graduation**: Defend the architecture against the Interlinker engineering review board and achieve Certified Interlinker Systems Architect status.

---

## 🏆 Evaluation & Milestone Checkpoints

| Milestone | Target Date | Criteria for Passing |
| :--- | :--- | :--- |
| **Milestone 1: Marten & Event Sourcing** | Day 15 | Complete 65 exercises in .NET. Pass all OCC and live aggregate tests. |
| **Milestone 2: Sagas & Outbox** | Day 23 | Complete routing slip saga with LIFO compensations and `SKIP LOCKED` outbox. |
| **Milestone 3: Rust Core Systems** | Day 39 | Complete all Rust borrow checker, traits, and async Tokio exercises with 0 compiler errors. |
| **Milestone 4: Native Rust Fabric** | Day 54 | Rebuild DocumentStore, Sagas, and Virtual Actors in Rust from scratch. |
| **Milestone 5: Interlinker Capstone** | Day 60 | Deploy the 500-exercise Capstone on Citus with 0ms UUIDv5 multi-tenancy and pass chaos tests. |

---

*Authored by Antigravity Autonomous Systems Engineering Directorate for the Interlinker Platform.*
