# Systems Design Interview: Distributed Key-Value Store

---

## Question

Your team needs to build an **internal distributed key-value store** — a general-purpose storage layer used by dozens of services across the company. Think: a simplified DynamoDB or Cassandra.

**Context and scale:**
- 10 billion keys at peak
- Average value size: 10 KB
- Read throughput: 1 million reads/sec
- Write throughput: 100K writes/sec
- p99 read latency target: 10ms
- p99 write latency target: 50ms
- 99.99% availability (four nines)
- Data must survive the simultaneous loss of any 2 nodes in a region
- Operations: `get(key)`, `put(key, value)`, `delete(key)` — no range queries required

You may assume a team of engineers to build and operate this. Design for production.

---

## Functional Requirements

> Fill this in. What does the system need to do? List the core behaviors.

```
- Handle writes
- Handle reads
- Fault Tolerant 
- highly available
- Handle Deletes
- Failed operations, i.e CRUD ops, and how to handle and flag to the system/user
- Faster reads than writes 
```

---

## Non-Functional Requirements / Scale Estimates

> Fill this in. What are the reliability, latency, throughput, and storage requirements? Work through the rough numbers — derive implications from the given scale, don't just repeat what I told you.

```
- I always struggle with this section because I feel like I address the main issues in functional requirements. 
```

---

## High-Level Design

> Sketch your initial architecture here. ASCII diagrams, component lists, data flow — whatever communicates your thinking. Cover the happy path end-to-end for a read and a write.

```
	[CLI] - [load balancer] - Write Path - [Leader] -> [replicate] -> [followers]
			  - Read Path  - [follower]
	[Replication] - [Lead Shard] - Update [Follower's shards]
	
	[Leader Election] - [I forget the algroithms for leader election]
```

---

## Buy-in

> Before going deeper, summarize the key decisions you want sign-off on. What consistency model are you targeting? How do you partition data? How do you replicate it? What trade-offs are you making?

_Candidate note: fill in the sections above, then write your buy-in summary here and submit._

```
- So the general idea is that we are going to have a classic Leader-Follower setup because we are seeing so much more reads than writes
- We will index on the keys but because there are so many keys we will have to shard the database.
- Replication is important because 2 nodes can be taken offline, lets say at anytime, all nodes need to be in sycnh
- There is synchonization challenges as well as split brain with leader election.
- We will have a load balancer to properly distribute load and maintain a cache to speed reads up 
- Cache invalidation will be tricky as there are still a decent amount of load and replication
- Because of read volume load balancer will just need to know traffice at a given DB and its status
- For Consistency model I think we might need to categorize data types. If a data classification being replicated is low/med then we can use an eventual model, however if we classify data as high pri/critical then we will provide different gurantees around data replication. In this case there would probably be two replication pipelines. **However I dont know alot about consistency models so that is a knowledge gap to add to study notes"
```

---

### Interviewer Buy-in Response

**Buy-in granted.** Move to the deep dive.

**On functional requirements:** Still thin. "Handle reads/writes/deletes" is just restating what I gave you. In a real interview you'd want to say things like: `get` on a missing key returns null (not an error); a `put` is acknowledged once written to a quorum of nodes; a `delete` uses a tombstone marker so replicas can propagate the deletion. You don't have to rewrite it — just know that this is the level of detail I'll expect when you explain your design.

**On the consistency model — this is the good part.** Your instinct to let the caller choose is exactly right, and it's how DynamoDB and Cassandra actually work. The mechanism isn't two separate replication pipelines — it's one replication mechanism with tunable quorums per operation. The concept is called **quorum reads/writes**: you have N replicas, a write succeeds when W of them acknowledge it, a read is satisfied when R replicas respond. The rule is R + W > N guarantees you'll always read your latest write. If you relax that (R + W ≤ N), you get eventual consistency but lower latency. The caller picks the trade-off per request. That's the model you were reaching for — hold onto it for the deep dive.

**Leader election:** You flagged this as a gap. The short version: **Raft** is the one to know. It works by having nodes elect a leader through a majority vote, and the leader maintains a replicated log. You don't need to know the internals cold, but you should be able to say "we use Raft for leader election, which requires a quorum of nodes to be available" and explain what happens when the leader dies.

---

**In the deep dive I want to see:**

1. **Partitioning** — how do you actually split 10B keys across nodes? How do you avoid hot partitions and handle adding/removing nodes?
2. **Replication and quorums** — make the N/R/W numbers concrete. Given the "lose any 2 nodes" constraint, what's your replication factor?
3. **Fault tolerance** — what happens when a node goes down mid-write? How does the system recover without losing data or double-applying a write?
4. **Storage engine** — what's actually on disk on each node? How do you handle deletes?
5. **Observability** — what does oncall watch?

Go deep.

---

## Deep Dive

> After receiving buy-in, expand on the details. Cover the hard parts: partitioning strategy, replication and quorums, fault tolerance, consistency guarantees, storage engine internals, and observability.

```
[your answer here]
```

---

## Interviewer Grading

> _(To be filled in by the interviewer after the candidate completes the Deep Dive.)_
