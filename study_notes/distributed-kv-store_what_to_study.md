# What to Study: Distributed Key-Value Store

## The Core Insight You Already Had

You instinctively said "categorize by data type and give different consistency guarantees." That's the right idea. The production name for it is **tunable consistency**, and it's the model Cassandra and DynamoDB use. Study this first — everything else in the deep dive flows from it.

---

## 1. Consistency Models and CAP Theorem

### CAP Theorem (the unavoidable foundation)
A distributed system can only guarantee two of three properties:
- **C**onsistency — every read sees the latest write
- **A**vailability — every request gets a response (not an error)
- **P**artition tolerance — the system works even when nodes can't talk to each other

Network partitions happen in production. So you're always choosing between C and A when a partition occurs. A KV store used by many teams likely favors **AP** (available + partition tolerant) with tunable consistency per request.

### Eventual Consistency
- Writes are accepted immediately on any available node
- Replicas sync asynchronously — a `get` may return stale data briefly
- Eventually all replicas converge to the same value
- Good for: session data, caches, feature flags, user preferences
- Trade-off: caller may read a write they just made and not see it (for a brief window)

### Strong Consistency
- A `get` always returns the latest `put`, no matter which node answers
- Achieved by requiring a quorum of nodes to agree before returning
- Good for: rate limiting counters, financial balances, anything where stale = wrong
- Trade-off: higher latency, unavailable during certain partitions

### Tunable Consistency (what you should propose)
Let the caller specify their consistency need per request. The mechanism is **quorums**.

---

## 2. Quorums — The Mechanism Behind Tunable Consistency

You have **N** replicas for each key. Every read and write involves a quorum:
- **W** = number of replicas that must acknowledge a write before it succeeds
- **R** = number of replicas that must respond to a read before returning

**The golden rule: R + W > N guarantees strong consistency.**

Why: if W nodes confirmed the write and R nodes must respond to the read, by the pigeonhole principle at least one node in the read quorum saw the write.

### Example with N = 3 (replication factor 3):
| W | R | R+W | Consistency | Trade-off |
|---|---|-----|-------------|-----------|
| 3 | 1 | 4 | Strong | Slow writes, slow if any replica down |
| 2 | 2 | 4 | Strong | Balanced |
| 1 | 3 | 4 | Strong | Fast writes, slow reads |
| 1 | 1 | 2 | Eventual | Fast everything, may read stale |

### For this problem:
"Survive loss of any 2 nodes" → you need **N = 3** minimum (lose 2, still have 1). To survive AND stay available during writes, N = 3, W = 2, R = 2 is the standard production choice. (Quorum = majority.)

---

## 3. Partitioning — How to Split 10B Keys Across Nodes

### Naive approach (wrong): Range partitioning
Split keys A-F on node 1, G-M on node 2, etc. Problem: hot partitions. If all traffic goes to keys starting with "user_", one node gets everything.

### Right approach: Consistent Hashing
1. Hash each key to a position on a virtual "ring" (0 to 2^32)
2. Each node owns a segment of the ring
3. A key is stored on the first node clockwise from its hash position
4. When a node is added/removed, only its neighboring keys need to move — not all keys

**Virtual nodes (vnodes):** Each physical node is assigned multiple positions on the ring (e.g., 150 virtual nodes per physical node). This:
- Smooths out load distribution (avoids one node getting a large arc)
- Makes rebalancing gradual when nodes join/leave
- Lets you weight nodes by capacity (a bigger machine gets more vnodes)

### How replication works with consistent hashing:
A key is replicated to the next N nodes clockwise from its position. So node failures don't require re-hashing — the next node in the ring is the natural replica.

---

## 4. Fault Tolerance

### Hinted Handoff
If a target replica node is down when a write comes in, a different node accepts the write temporarily and stores a "hint" — "deliver this to node X when it comes back online." Keeps writes available even during node failures.

### Anti-Entropy / Read Repair
Over time, replicas can drift (failed syncs, network hiccups). Two mechanisms fix this:
- **Read repair:** When a read detects replicas disagree, it writes the latest value back to stale replicas as a side effect
- **Anti-entropy (Merkle trees):** Background process compares hashes of data between replicas to find and fix divergence without transferring all data

### Gossip Protocol
Nodes broadcast their state (alive/dead, load, ring position) to a few random neighbors periodically. Within a few rounds, all nodes know all state — no central coordinator needed. This is how Cassandra does failure detection.

---

## 5. Storage Engine — What's on Disk

### LSM Tree (Log-Structured Merge Tree) — used by Cassandra, RocksDB
1. Writes go to an in-memory buffer (memtable) — very fast
2. When the buffer is full, it's flushed to an immutable sorted file on disk (SSTable)
3. Reads check the memtable first, then SSTables from newest to oldest
4. Background compaction merges SSTables to reclaim space and improve read speed

**Why LSM for a KV store:**
- Writes are sequential (fast on SSD and HDD)
- Optimized for write-heavy workloads (100K/sec fits well here)
- Trade-off: reads can be slower without a bloom filter

**Bloom filter:** A space-efficient probabilistic structure that answers "is this key definitely NOT in this SSTable?" — lets you skip SSTables that can't contain the key, dramatically speeding up reads.

### Deletes — Tombstones
You can't just remove a key from an SSTable (it's immutable). Instead, `delete(key)` writes a special marker called a **tombstone**. During reads, a tombstone means "this key is deleted." During compaction, tombstones eventually purge the original data.

---

## 6. Leader Election — Raft (the one to know)

Raft is the standard algorithm for electing a leader and maintaining a replicated log. The short version:

1. Nodes start as **followers**. If they don't hear from a leader, they become a **candidate** and request votes.
2. A candidate becomes **leader** if it gets votes from a **majority** (quorum) of nodes.
3. The leader receives all writes, appends them to its log, and replicates to followers.
4. A write is committed only when a majority of followers have acknowledged it.

**Why quorum matters for safety:** A leader can only be elected with majority votes. A write is only committed with majority acknowledgment. So any two quorums overlap — meaning no two leaders can exist simultaneously, and no committed write can be lost.

**In practice:** Most teams don't implement Raft from scratch. They use ZooKeeper or etcd as the coordination service for leader election, and build the KV store on top.

---

## 7. Observability — What Oncall Watches

Four golden signals applied to a KV store:
- **Latency:** p99 read and write latency per node and per shard
- **Replication lag:** how far behind are follower replicas? (key metric — if lag spikes, reads are stale)
- **Error rate:** failed reads/writes, quorum failures (couldn't reach W/R nodes)
- **Saturation:** disk usage per node, memtable size, compaction backlog

**Operational alerts to add:**
- Node marked down by gossip → page oncall
- Replication lag > threshold → risk of stale reads
- Compaction falling behind → disk filling, reads slowing
- Quorum failures → availability degrading

---

## 8. What a Strong Deep Dive Answer Looks Like

For this specific problem, the right order to go through the components:

### Step 1: Partitioning
"We use consistent hashing with virtual nodes. Each key is hashed to a position on the ring. Each physical node owns ~150 virtual positions. A key is stored on the next N=3 nodes clockwise. Adding/removing a node only moves its neighboring key ranges."

### Step 2: Replication and Quorums
"Replication factor N=3, chosen because we need to survive loss of any 2 nodes and still have 1 available. For default writes: W=2 (quorum). For default reads: R=2 (quorum). R+W=4 > N=3, so strong consistency is guaranteed. Callers can request R=1 for low-latency eventual consistency reads."

### Step 3: Fault Tolerance
"If a replica is down during a write, we use hinted handoff — a neighbor accepts the write and delivers it when the node recovers. Background anti-entropy using Merkle trees detects and repairs divergence. Failure detection runs via gossip protocol — no central coordinator."

### Step 4: Storage Engine
"Each node runs an LSM tree. Writes hit the memtable first, flushed to SSTables on disk. Reads check memtable then SSTables, using bloom filters to skip irrelevant files. Deletes write tombstones. Background compaction merges SSTables."

### Step 5: Leader Election
"For coordinator HA, we run Raft via etcd. Writes are coordinated by the current leader. If the leader dies, Raft elects a new one within seconds — a follower that has heard from a majority becomes the new leader."

### Step 6: Observability
"Track replication lag, p99 read/write latency, quorum failure rate, and disk saturation per node. Dead-letter any write that fails after N retries. Gossip state is the source of truth for node health."

---

## Quick Reference: The Numbers for This Problem

| Dimension | Estimate | Implication |
|-----------|----------|-------------|
| Total storage | 10B keys × 10KB = **100 TB** | ~20-30 nodes at 4-6 TB usable SSD each, before replication |
| With replication (N=3) | 100 TB × 3 = **300 TB raw** | ~60-100 nodes |
| Write throughput | 100K/sec × ~1KB metadata = modest | LSM tree handles this easily |
| Read throughput | 1M/sec across ~100 nodes = **10K reads/node/sec** | In-memory bloom filters + memtable keep this fast |
| Failover speed | Four nines = 52 min/year downtime | Raft leader election < 10 seconds; node replacement is the bottleneck |
