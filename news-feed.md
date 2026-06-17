# System Design Interview: Social Media News Feed

## Question

Design a social media news feed (Facebook/Twitter style). 500M daily active users, mostly reads. Users can follow other users and see a personalized feed of posts when they open the app.

---

## Functional Requirements

- Users can create posts (text + images, ~1KB metadata, images in blob storage)
- Users can like posts
- Users see a feed of posts from people they follow
- Feed is ordered by arrival time (no strict ranking enforced)
- Feed supports infinite scroll with pagination; feed stops when posts run out
- Comments and shares out of scope

---

## Non-Functional Requirements & Scale

| Metric | Value |
|---|---|
| DAU | 500M |
| Write rate | ~6k posts/sec (500M/day) |
| Read:write ratio | ~50:1 |
| Read rate | ~300k feed reads/sec |
| Post size | ~1KB metadata |
| Feed load latency | < 1 second |
| Post propagation SLA | 5–10 seconds |
| Consistency model | High availability, eventual consistency |

---

## High-Level Design

```
Write path:
Client → API Gateway → Load Balancer → Post Service → Message Queue → Fan-out Workers → Cache + DB

Read path (regular users):
Client → API Gateway → Load Balancer → Feed Service → Feed Cache (precomputed) → merge → response

Read path (celebrity posts):
                                                      └→ Celebrity Post Cache (direct read) → merge → response
```

**Key components:**
- **Post Service**: accepts post creation, writes to DB, publishes to message queue
- **Message Queue**: decouples post creation from fan-out; absorbs write spikes
- **Fan-out Workers**: consume queue, look up follower list, write post ID to each follower's feed cache
- **Feed Cache**: Redis key-value store; key = `feed:user:{id}`, value = ordered list of post IDs
- **User-to-cache routing table**: maps each user ID to their home cache node (consistent hashing)
- **Celebrity Post Cache**: heavily replicated read cache for posts from high-follower users; queried directly at read time

---

## Design Decisions

### Fan-out on Write (regular users) vs Fan-out on Read (celebrities)

Regular users (< ~10k followers): fan-out on write. Fan-out workers precompute each follower's feed cache entry. Feed reads are O(1) cache lookup.

Celebrities (millions of followers): fan-out on write is too expensive (1M+ cache writes per post). Instead, celebrity posts live in a dedicated, heavily replicated cache. At read time, the feed service merges the precomputed regular feed with a direct synchronous query to the celebrity cache.

**Celebrity threshold**: determined by follower count at write time. User metadata includes a flag for celebrity status.

### Inactive User Optimization

Fan-out workers skip cache writes for inactive users (haven't opened app in N days). Use activity tiers (ACTIVE / PASSIVE / INACTIVE) with different TTLs:
- ACTIVE: short TTL, frequent cache refresh
- PASSIVE: medium TTL
- INACTIVE: no precomputed feed; computed on demand on next login

### Cursor-based Pagination

Feed pagination uses post ID as cursor, not numeric offset. Each request: "give me 20 posts older than post_id X." This is stable across cache rebuilds, post deletions, and concurrent new arrivals. New posts during an active scroll session are appended to the end of the in-progress feed to avoid disrupting the user's position.

### Hotspot Mitigation (Celebrity Posts)

A single celebrity post can receive 500M read requests in minutes. The celebrity post cache is replicated across many nodes so read load is distributed. No single node is the bottleneck for a viral post.

---

## Grading

| Dimension | Score | Notes |
|---|---|---|
| Functional Requirements | 4/5 | Solid. Post creation, feed, likes, pagination covered. Missed explicitly framing read-heavy nature upfront. |
| Scale & NFRs | 3/5 | Got there with prompting. Initial read estimate was 3x writes — needed to be walked to 50:1 and 300k/sec. |
| High-Level Design | 4/5 | All core components present. Needed a nudge toward the message queue but landed on it quickly. |
| Deep Dive | 3/5 | Good instincts on activity TTL tiers and cursor pagination but needed guidance on offset-vs-cursor distinction and celebrity hotspot solution. |
| Communication | 4/5 | Thought out loud well, flagged uncertainty honestly, self-corrected multiple times. |
| **Overall** | **3.5/5** | |

---

## Model Answer Notes

The two canonical approaches to news feed delivery:
- **Fan-out on write (push)**: precompute feeds at post time. Fast reads, expensive writes for high-follower users.
- **Fan-out on read (pull)**: compute feed at request time. Simple writes, expensive reads at scale.
- **Hybrid** (what you described): fan-out on write for regular users, fan-out on read for celebrities. This is what Facebook and Twitter use in production.

The celebrity threshold is typically ~10k–100k followers depending on system scale.

---

## Study Notes

### Key Concepts to Review

**Fan-out patterns**
- Fan-out on write: precompute each follower's feed at post time via async workers
- Fan-out on read: compute feed on demand by querying all followed users
- Hybrid: the standard production approach — know when to switch strategies

**Feed cache structure**
- Redis sorted set or list per user: `feed:user:{id}` → list of post IDs
- Actual post content fetched separately (post ID → post content lookup)
- Separating feed ordering from post content makes cache invalidation cheaper

**Cursor-based pagination**
- Cursor = last seen post ID, not a numeric offset
- Stable across inserts, deletes, and cache rebuilds
- Industry standard for any feed/timeline system

**Hotspot / celebrity problem**
- A single entity receiving disproportionate traffic (hot key)
- Solutions: dedicated replicated cache, read replicas, CDN for static content
- At interview time: identify the hot key, then describe how you isolate and replicate it

**Back-of-envelope estimation**
- Social media is typically 50:1 to 100:1 reads:writes
- DAU × actions/day / 86400 = requests/second
- Always sanity-check your ratio against real product behavior before committing to a number

**Consistent hashing**
- Used to map user IDs to cache nodes
- Allows cache nodes to be added/removed with minimal key remapping
- Review: virtual nodes, hash ring, rebalancing

### Derived Numbers for This Problem

| Signal | Value |
|---|---|
| Write QPS | ~6k/sec |
| Read QPS | ~300k/sec |
| Daily post data | ~500GB/day |
| Feed cache entry | list of ~50–100 post IDs per user |
| Fan-out writes per post (avg) | ~500 followers × 6k posts/sec = ~3M cache writes/sec |
