# What to Study: Distributed Job Scheduler

## Topics to Cover

### 1. Back-of-Envelope Estimation
A learnable skill. Study byte sizes (KB/MB/GB/TB), common latencies (memory vs disk vs network), and how to chain them into storage and throughput estimates.

**Practice pattern:**
- Record size × record count = storage
- Requests/sec × average latency = concurrent workers needed
- Work this through for every system before designing it

**Resource:** *Systems Design Interview* by Alex Xu, Vol. 1 — chapter on estimation.

---

### 2. Time-Based Indexing and the Scheduler Pattern
Your instinct (priority queue sorted by execution time) was correct. Study the mechanics.

**Key concepts:**
- B-tree indexes and range queries: `WHERE execution_time BETWEEN NOW() AND NOW() + 1s`
- Redis sorted sets (`ZRANGEBYSCORE`): in-memory, sub-millisecond lookup by score (timestamp)
- The "tick loop" pattern: a process that wakes every N ms, queries for due work, dispatches it

**Production examples to look at:** Celery Beat, Quartz Scheduler — both implement this pattern.

**The two viable approaches:**
1. DB index on `execution_time` — simple, correct, works up to moderate scale
2. Pre-load upcoming jobs into a Redis ZSET — decouples hot-path trigger loop from DB, better for high throughput

---

### 3. Exactly-Once Execution
The hardest constraint in this problem. Two workers must not both fire the same job.

**Key concepts:**
- Compare-and-swap (CAS) on job status: only one writer can move `PENDING → RUNNING` — the loser gets a write conflict and backs off
- Distributed locks: Redis `SET key value NX PX <ttl>` — only one holder at a time
- Idempotency keys: even if a job fires twice, the receiver can detect and deduplicate

**The pattern:** claim the job with a CAS update first, then fire the callback. If two workers race, only one wins the CAS.

---

### 4. Fault Tolerance and Leader Election
A single scheduler node = SPOF. Study how to make it HA without causing double-fires.

**Key concepts:**
- Active/standby with leader election: only one scheduler is active at a time; ZooKeeper or etcd holds the lock
- Partitioned active/active: split the job namespace (e.g., by hash of job ID) across N scheduler instances — each owns a non-overlapping slice
- Why naive "just run two schedulers" breaks: both find the same due jobs and both fire them

**The link to NFRs:** "must not miss jobs" → scheduler must be HA → scheduler cannot be a single node.

---

### 5. Clock Skew
Brief but important for the deep dive.

**Key concepts:**
- Wall clocks on different machines drift — NTP syncs them but doesn't eliminate drift
- A scheduler whose clock is 2 seconds behind will fire jobs late; 2 seconds ahead will fire early
- Mitigations: use a single authoritative time source, build in a small look-ahead window, don't rely on exact wall-clock equality across nodes

---

### 6. Observability
For any production system, know the four golden signals: latency, traffic, errors, saturation. For a scheduler specifically:

- **Lag metric:** how far behind is the scheduler? (current time - most recently fired job's scheduled time)
- **Dead-letter queue:** where do jobs go after N failed retries?
- **Job status tracking:** PENDING → RUNNING → COMPLETED/FAILED — queryable by callers

---

## How to Go Deeper in a Design Interview

The move from basic design to deep dive is: **pick your scariest component and break it.**

For each component, ask three questions:

1. **What happens when this fails?**
   - Scheduler dies → jobs missed → need redundancy + leader election
   - Worker dies mid-execution → job half-done → need CAS on status, retry from PENDING
   - Queue backs up → trigger latency blows past 1s → need backpressure, more workers

2. **What breaks at 10x scale?**
   - Single DB → can't handle 10M triggers/sec → shard by time bucket or job ID
   - Single scheduler → bottleneck → partition job namespace across scheduler instances

3. **How do you satisfy each hard constraint explicitly?**
   - "No double-fire" → CAS on job status, distributed lock
   - "No missed jobs" → durable job store, at-least-once dispatch, idempotency on receiver

**For this problem, the right deep dive order would be:**
1. Scheduler as SPOF → how do you make it HA without double-firing?
2. How do you find due jobs efficiently? (the index / Redis ZSET answer)
3. Exactly-once given network failures? (CAS + idempotency keys)
4. How do you shard the job store so one partition isn't a hotspot?

---

## Applied to This Problem: The Numbers

| Dimension | Estimate | Implication |
|-----------|----------|-------------|
| Job record size | ~1 KB (id, timestamp, URL, payload, status) | 50M jobs = **~50 GB** — one Postgres instance handles this |
| Trigger throughput | 100K/sec | Kafka handles this easily; your worker pool is the bottleneck |
| Worker concurrency | 100K triggers/sec × 100ms avg callback = **10K concurrent workers** | Worker pool must be large; probably need horizontal scaling |
| Availability target | 99.99% (four nines) = ~52 min downtime/year | Scheduler must have redundancy — no single node |
| Historical job data | 7M jobs/day × 365 = ~2.5B rows/year | Need archival strategy after ~6 months |
