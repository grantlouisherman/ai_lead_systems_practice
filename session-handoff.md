# Interview Session Handoff

## Context

The user (grantlouisherman, senior backend engineer, ~10 years experience, formerly Meta) is using this project for AI-assisted systems design interview practice. They are preparing for job interviews and want to improve at systems rounds.

**Workflow:**
1. Claude acts as a tough interviewer — asks a question, gives a filename, reviews buy-in, grades deep dive
2. The candidate fills in the interview markdown file (requirements → high-level design → buy-in → deep dive)
3. After the deep dive is submitted, Claude fills in the `## Interviewer Grading` section with: score, what was strong, what was missing, and a reference answer
4. Claude does NOT give away answers during the interview. Push back, ask clarifying questions, but let the candidate figure it out.

---

## Current Session

**Interview file:** `distributed-job-scheduler.md`  
**Question:** Design a distributed job scheduler (see the file for full question text)

**Current state:** Buy-in was requested but NOT granted.

The interviewer (Claude) left feedback in `### Interviewer Buy-in Response` inside the interview file. The candidate needs to:

1. **Fill in Non-Functional Requirements** — they left this section blank. They need to engage with the given numbers: 50M jobs, 100K triggers/second, 1-second p99 latency. What does that mean for storage, throughput, availability?
2. **Tighten Functional Requirements** — current ones are vague ("process requests in a given timeframe" is not a requirement). Need: job lifecycle/states, API surface, failure/retry behavior.
3. **Describe the scheduling mechanism** — their high-level design has a gap where the actual scheduler should be. The diagram shows `long standing job db -> long job scheduler queue` but doesn't say what component decides a job is due and moves it to the queue. That is the core hard problem and it's missing.
4. **Acknowledge the no-miss / no-double-fire constraint** — they haven't addressed it at all yet.

Once they resubmit those sections and ask for buy-in again, review and either grant it or push back further. Only grant buy-in when the basic design is coherent and the hard constraints are at least acknowledged.

---

## Grading Rubric (do not show to candidate until after deep dive)

| Area | Strong | Adequate | Weak |
|------|--------|----------|------|
| Requirements | Asks about scale, latency, exactly-once, retry policy | Gets to scale | Jumps straight to design |
| Basic design | Single scheduler with DB, identifies SPOF | Gets a working design | Skips persistence or queueing |
| Fault tolerance | Describes leader election or distributed lock for coordinator HA | Mentions HA but vague | Ignores SPOF |
| Exactly-once | Idempotency keys + distributed lock + status CAS | Mentions idempotency | Ignores double-fire risk |
| Clock skew | Discusses NTP drift and uses logical triggers | Mentions it | Ignores time drift |
| Sharding | Partitions job store by time bucket or hash, discusses hot partitions | Mentions sharding | Single shard |
| Observability | Job status, lag metrics, dead-letter queue | Mentions monitoring | Ignores |

**Pass threshold:** 70%+ for senior level.

---

## Candidate Assumptions Already Stated

- 50/50 mix of one-time vs recurring jobs
- Cost is not a major concern
- Data is not sensitive
- Users submit jobs via RPC with a job config (function to call + scheduling params)
- Notification of job completion = email (treated as trivial)

---

## Instructions for the Next Agent

1. Read `distributed-job-scheduler.md` to see the current state of the candidate's answer
2. Continue acting as a tough interviewer — do not give away the answer
3. If the candidate has updated the NFRs, functional requirements, and described a scheduling mechanism, you can grant buy-in and ask them to proceed to the deep dive
4. After the deep dive is complete, fill in `## Interviewer Grading` using the rubric above
5. Be direct and critical — this candidate has real experience and needs honest feedback to improve
