# Systems Design Interview: Distributed Job Scheduler

---

## Question

Your company runs a platform used by thousands of internal teams. They need a **distributed job scheduler** — a service that lets callers register jobs (one-time or recurring) and guarantees those jobs are executed at the right time.

**Context and scale:**
- 50 million registered jobs at any given time
- Peak scheduling load: 100,000 job triggers per second
- Jobs can be one-time ("run at 2026-07-01 09:00 UTC") or recurring ("every 5 minutes")
- The smallest scheduling granularity is 1 second
- Job execution is a callback — the scheduler calls a provided HTTP endpoint or publishes to a queue topic; it does not run user code directly
- Acceptable trigger latency: a job should fire within 1 second of its scheduled time (p99)
- The system must not miss jobs and must not double-fire them

You may assume a team of engineers to build and operate this. Design for production.

---

## Functional Requirements

> Fill this in. What does the system need to do? List the core behaviors.
- User requests executed within a given time frame
- Consistency in terms of jobs not being missed or done twice
- low latency in terms of submission and scheduled but are executing every 5 mins
- Fairly high load and pretty large memory requirement in terms of "registered jobs"
- SLA is around when job will be executed but not around when it will be completed
---

## Non-Functional Requirements / Scale Estimates

> Fill this in. What are the reliability, latency, throughput, and storage requirements? Work through any rough numbers.

- Peak load is 100k per sec, but rough number is like 7-8 million jobs submitted per day
- In terms of storage we will need to store long standing jobs as well as caches for results of RPC
- Reliability is around execution of jobs.Job could fail mid write or mid execution we would need to handle that on our end. Do we scrap and start over? We dont know how long an average job is so that is an open question to answer.
- How do we handle errors, we can have retries on our end but how can we handle failures, part completion, etc.

---

## High-Level Design

> Sketch your initial architecture here. ASCII diagrams, component lists, data flow — whatever communicates your thinking. Aim to cover the happy path end-to-end.

```
struct Job {id, execution_time, is_cont}
Basic Path: [CLI] -> [load balancer] -> [Message Queue] -> [fast|med|slow] -> [Worker] -> [Sends back results]
Long standing jobs: [Basic Path] - concurrently - [Save job meta data into DB] - [Scheduler pulls Jobs] and then [Basic Path]
```

---

## Buy-in

> Before going deeper, write a brief summary of the key decisions you want sign-off on. What assumptions are you locking in? What trade-offs are you making? This is where you check in with the interviewer.

_Candidate note: submit the file once you've filled in the sections above and are ready for buy-in feedback. The interviewer will respond in the section below._

Here are some of my assumptions:
- There will be 50/50 mix of one time jobs and continous jobs
- Cost is not a major concern
- Outside normal security the data is not sensitve so the results can just be sent to the user
- Our system owns the code so there is not fear of remote execution attacks
- Notifiying a user is a trival aspect of this and I can imagine that I just send them an email
- Assuming a user is going to use one of our RPCs and they are just giving us the inputs
Basic Plan:
Right now my basic idea is that we would have a CLI where a user submits a jobs
There would be two basic customer facing interfaces which would be oneTimeJob and longStandingJob
	- oneTimeJob happens once 
	- longStanding job is something that would live on our end and have to be handled in its own database

Users are able to make the a CLI request with a JOB config that gives the specifics of what function needs to be called and then our system knows how to handle the longer term jobs. There would be a flag in the config that mentions if we need to do this continously.

In terms of high level design we would do something like CLI -> [load balancer] -> [queue servers]. Then workers pull jobs off of the queue and perform the operations. For long standing jobs we would have [long standing job db] -> [long job scheduler queue] => and then this would feed into our load balancer and then just start

In the message queue we will need to prioritize jobs on when they need to be completed by, that can be apart of our SLAs. We would need to respect prioity in terms of time because we have a 1 second of a given time to execute. So we might need to resort every second which would be really exspensive. There is an ability to try and categorize queue items in different lanes, i.e fast, medium, and slow. Since we have such an aggressive execution time we wouldnt need to handle a bunch of resorts but obviously then we have the extra logical and possible issues where if a worker always priortizes the fast lane, there is a possiblity that the fast lane gets bogged down and starves the other queues.

So there is a trade off in terms of some complexity. I think the way that I would handle this is that I am good with having the different lanes of scheduled time, but I would start to rate limit if the given queue is getting too big. I.e fast lane jobs are super backed up we are rate limiting users with a cool off. 

For long standing jobs there is the issue around ho 
--- 

### Interviewer Buy-in Response (Round 2)

Still not ready. You made some progress — the queue lane prioritization thinking is reasonable and I like that you're grappling with the 1-second window. But three of the four issues from last round are still open:

**1. NFRs still have no actual numbers.**
"Peak load is 100K/sec" is just re-reading what I told you. I need *you* to derive the implications. If each job record is, say, 500 bytes, what does 50M registered jobs cost in storage? What write throughput does 100K triggers/sec demand from your queue? What happens to the system if the scheduler goes down — do you target 99.9% or 99.99% availability, and why does that matter for your design? Work the numbers. Rough estimates are fine, but show me the reasoning.

**2. Functional requirements still aren't functional requirements.**
Your list mixes NFRs ("fairly high load"), vague goals ("consistency"), and non-requirements ("SLA is around when job will be executed"). I need behaviors. What states does a job move through — PENDING → SCHEDULED → RUNNING → COMPLETED/FAILED? What APIs does a caller use: submit, cancel, get-status? What happens when the callback endpoint returns a 500? These aren't optional details — they define the contract your system has to keep.

**3. The scheduler is still a black box.**
You wrote `"Scheduler pulls Jobs"` in your diagram. That is not a design. I need to know: how does the scheduler find jobs that are due? You have 50 million job records in a database. Every second, some subset of them become due. How does the scheduler identify which ones efficiently without doing a full table scan? This is the single most important question in this problem and you still haven't touched it. I'm not asking you to solve all the edge cases — just tell me at a high level what the mechanism is.

**What you got right:** The fast/medium/slow lane idea is a reasonable way to reduce priority churn. Rate-limiting submissions when a lane is saturated is pragmatic. That thinking belongs in your design. Keep it.

Fix items 1–3 and come back.

---

## Deep Dive

> After receiving buy-in, expand on the details. Cover the hard parts: fault tolerance, exactly-once execution, scaling the scheduler itself, handling clock skew, job storage and indexing, and observability.

```
[your detailed design here]
```

---

## Interviewer Grading

> _(To be filled in by the interviewer after the candidate completes the Deep Dive.)_
