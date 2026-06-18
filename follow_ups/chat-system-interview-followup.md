# Follow-Up: Chat System Design Interview

**Date:** 2026-06-17  
**Topic:** Design a messaging system at WhatsApp scale

---

## Scale Parameters (memorize these)

| Parameter | Value |
|---|---|
| Registered users | 2 billion |
| Daily active users | 500 million |
| Messages per day | 100 billion |
| Messages per second (peak) | ~1.15 million |
| Avg message size | ~1 KB |
| Storage per day | ~100 TB |
| Storage per year | ~36 PB |
| Max group size | 1,000 members |
| Delivery latency target | 1–2 seconds |

**Drill these derivations until they're automatic.** In a real interview you should have storage/throughput numbers on the board within the first 5 minutes, unprompted.

---

## Core Architecture

```
Client A
   |
   | (WebSocket)
   |
Chat Server 1 ──────────────── Redis (routing table: userId → serverId)
   |                                      |
   |                              Chat Server 2 ──── Client B (WebSocket)
   |                                      |
   └──────────── Kafka ───────────────────┘
                    |
              Message Store (Cassandra)
```

### Key Components

**WebSocket Servers**
- Every active user holds one persistent WebSocket connection to a chat server
- At 500M DAU with 100K connections/server → ~5,000 servers needed
- Servers are stateless except for active connections — all durable state lives elsewhere

**Routing Table (Redis)**
- Maps `userId → chatServerId`
- Written on connect, deleted/expired on disconnect
- Use TTL so stale entries auto-expire if a server crashes (server refreshes TTL while alive)
- Redis Cluster with consistent hashing to shard at scale
- Sub-millisecond lookup — this is the fast path for every message delivery

**Kafka (Message Broker)**
- Decouples senders from receivers across the server fleet
- Chat Server 1 publishes message → Kafka routes to Chat Server 2 → Server 2 pushes via WebSocket to recipient
- Also provides durability: messages survive a server crash mid-delivery

**Cassandra (Message Store)**
- Partition key: `conversation_id`
- Clustering key: `timestamp` (descending)
- Primary query: "give me last N messages in conversation X" — maps perfectly to this schema
- `seen` boolean column for read receipt tracking
- Horizontally scalable, no joins, fast sequential reads

---

## Fan-Out Strategy (Hybrid)

| Group size | Strategy | Reason |
|---|---|---|
| < 500 members | **Push** (fan-out on write) | Manageable write amplification, real-time delivery |
| ≥ 500 members | **Pull** (client fetches on load/scroll) | Avoids 1,000× write amplification per message |

**The math that drives this decision:**  
1M messages/sec × 1,000 members × 1,000 routing lookups = 1 billion operations/sec if you push everything. That's untenable. Pulling for large groups trades slight latency for linear write cost.

Same hybrid pattern applies to **read receipts** — individual receipts per member in a large group don't scale; aggregate ("X of 1000 read") via polling is both cheaper and better UX.

---

## Storage Tiering

- **Hot (Cassandra):** Last ~90 days. Drives the cutoff with real access pattern data — what % of reads are older than 30/60/90 days?
- **Cold (S3/object store):** Older messages, archival. Cheaper per GB, higher latency on retrieval.
- At 36 PB/year you cannot keep everything in hot storage — tiering is required, not optional.

---

## Failure Modes

| Failure | Impact | Recovery |
|---|---|---|
| Chat server crash | Active connections dropped | Clients reconnect to new server; routing table entry expires via TTL; client fetches undelivered messages from Cassandra on reconnect |
| Redis node down | Routing lookups fail | Redis Cluster failover; in-flight messages can be retried via Kafka |
| Kafka broker down | Message delivery stalls | Kafka replication across brokers; producers retry with backoff |

---

## What to Lead With Next Time

1. **Scope first** — you did this well, keep it
2. **Derive numbers immediately** — write throughput, storage/day, storage/year on the board before any architecture
3. **State your access patterns before picking a database** — "primary query is X, therefore Y database" not "I like SQL/NoSQL"
4. **Close the loop on big numbers** — when you derive a scary number (1B fan-out ops/sec), immediately say whether it's a problem and what you'd do about it
5. **Drive, don't wait** — propose the next component before the interviewer asks. You should be saying "now let's talk about fan-out" not waiting to be led there

---

## Things That Went Well

- Asked for scale parameters before designing (strong start)
- Identified websockets vs polling tradeoff correctly
- Got to hybrid fan-out strategy independently
- Recognized tiered storage need from the 36PB number
- Honest about gaps (Redis routing) rather than bullshitting

---

## Concepts to Review

- **Redis TTL-based presence/routing** — how chat apps track which server a user is on
- **Cassandra data modeling** — partition keys, clustering keys, time-series patterns
- **Kafka consumer groups** — how multiple chat servers consume from the same topic
- **WebSocket server architecture** — connection limits, horizontal scaling, sticky sessions vs stateless
- **Fan-out on write vs read** — when each is appropriate (also covered in news-feed notes)
