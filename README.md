# TechClarity — Architecture Decision Record

> Every component in this system exists for a reason. This document explains the *why* behind each decision — the tradeoffs, the failure modes we designed around, and the reasoning that shaped the architecture.

---

## The Core Philosophy

**Reliability over simplicity. Async over blocking. Horizontal over vertical.**

AI generation is slow. Databases fail. Servers crash. This architecture assumes all three will happen and designs around them — not with hope, but with redundancy, decoupling, and graceful degradation.

---

## Why an Application Load Balancer?

A single server with a public IP is a liability. The ALB is the only publicly exposed surface of the entire system.

**What it gives us:**
- TLS termination in one place — API instances never touch raw HTTPS
- Health-check-based routing — a crashed API instance gets zero traffic within seconds
- The foundation for horizontal scaling — add instances behind it without touching DNS

> *Without the ALB, a single API server crash takes down the entire product. With it, the fleet absorbs the failure silently.*

---

## Why Auto Scaling Group (ASG) and not a single server?

A single server is a **Single Point of Failure**. One memory leak, one bad deploy, one hardware fault — everything is down.

The ASG means:
- **Fault tolerance** — instances fail independently. Two unhealthy out of five still leaves three serving traffic.
- **Scale on demand** — a traffic spike at 2pm doesn't require manual intervention
- **Zero-downtime deploys** — rolling replacements, not "pray during `git pull`"

> *Think of it like running three tills at a supermarket instead of one. If one breaks, the queue moves. It doesn't stop.*

The API instances are **stateless by design** — no session data lives on them. That's intentional. Stateless instances can be killed and replaced without any user noticing.

---

## Why Redis for sessions?

Because stateless API instances need *somewhere* to share state.

If session data lived in memory on API #1, a request hitting API #2 would see the user as logged out. Redis acts as the shared brain — fast, in-memory, and accessible by every instance in the cluster.

**The secondary use case is equally important:** job status keys. When a client polls `/jobs/{id}/status`, the API reads a single Redis key rather than querying DocumentDB. Sub-millisecond reads, zero DB load.

> *Redis is the reason we can have ten API instances that all behave like one.*

---

## Why SQS? Why async processing at all?

PRD generation via an LLM takes **15–90 seconds**. Holding an HTTP connection open for 90 seconds is:

- Fragile (network timeouts, mobile drops)
- Wasteful (a thread blocked for 90s is a thread not serving other users)
- Unscalable (100 concurrent users = 100 blocked threads)

With SQS, the API does two things: write the job, return the job ID. Done in under 100ms. The client polls for status. The AI worker processes in its own time, on its own instance.

**This decoupling means:**
- API performance is independent of AI generation speed
- A spike in generation jobs doesn't slow down authentication or project loading
- Failed jobs can be retried without the user ever knowing

> *A restaurant doesn't make you stand at the counter while your food is cooked. They give you a ticket and call your number. SQS is the ticket.*

---

## Why a dedicated AI Worker instead of running generation inside the API?

The API cluster is optimised for **fast, concurrent, short-lived requests**. AI generation is the opposite — slow, CPU/GPU-intensive, long-running.

Mixing them means:
- A generation task monopolises an API instance for 60+ seconds
- Other users experience latency for requests that should be instant
- You can't independently scale generation capacity from request-handling capacity

The AI Worker runs on a separate EC2 instance (GPU-capable when needed), scales independently, and can be upgraded without touching the API fleet.

> *Separation of concerns at the infrastructure level, not just the code level.*

---

## Why DocumentDB with a Primary and Replica?

**The replica exists to protect the primary.**

In a read-heavy SaaS, the majority of database operations are reads — fetching projects, loading PRDs, rendering dashboards. Sending all of that to the primary wastes write capacity and increases the risk of the primary becoming a bottleneck.

- **Primary** — handles all writes. Authoritative source of truth.
- **Replica** — handles read traffic. If it falls behind or fails, reads degrade gracefully; writes are unaffected.

> *A library with one copy of every book is a single point of failure for readers. The replica is the second copy.*

The deeper reason: **replication is cheap insurance**. The cost of running a replica is a fraction of the cost of an outage caused by a primary under unsustainable read load.

---

## Why Pinecone (Vector Database)?

Prompting an LLM with raw user input produces generic output. Prompting it with *relevant context from the user's own project history* produces specific, high-quality output.

Pinecone stores **embeddings** — numerical representations of documents, features, and previous PRDs. When a new generation job runs, the AI Worker queries Pinecone for semantically similar context and injects it into the prompt.

This is **Retrieval-Augmented Generation (RAG)**. Without it, the LLM has no memory of the product it's generating for.

> *The difference between an LLM that writes a generic PRD and one that writes *your* PRD is context. Pinecone is how we store and retrieve that context at scale.*

---

## Why S3 for PRD storage?

Generated PRDs are structured documents — potentially large, infrequently mutated, frequently downloaded. That is exactly the problem object storage solves.

Storing documents in DocumentDB would:
- Balloon row sizes and degrade query performance
- Make large document retrieval slow and expensive
- Complicate versioning

S3 gives us **cheap, durable, infinitely scalable storage** with signed URL delivery — the client downloads directly from S3, not through the API. The API never touches the document bytes.

> *The database stores where the document is. S3 stores what the document is.*

---

## Why Sentry?

Because `console.log` is not a production monitoring strategy.

Sentry captures unhandled exceptions with full context — stack trace, request payload, user ID, environment, and a timeline of what happened before the crash (breadcrumbs). When API #3 throws a `TypeError` at 3am, Sentry has the full story before anyone opens a terminal.

**The specific value:** Sentry groups duplicate errors. If the same bug fires 400 times in an hour, you get one alert with a counter — not 400 notifications.

> *You don't know your application is broken until users tell you, unless you have Sentry.*

---

## Why PagerDuty?

Sentry tells you *what* broke. PagerDuty ensures a **human actually responds**.

In a team with on-call rotations, PagerDuty manages who gets woken up, in what order, with what escalation path. Silence an alert for too long and it automatically escalates to the next engineer.

At current scale — a small team — Sentry → Slack is sufficient. PagerDuty becomes essential when there are **SLAs with paying customers** and "someone will see it in the morning" is no longer acceptable.

> *The distinction: Sentry is a diagnostic tool. PagerDuty is a accountability tool.*

---

## Why OpenVPN Bastion?

Private subnet instances have no public IP. That is intentional. The only way in is through the bastion.

**What this prevents:**
- Direct SSH brute-force against database and worker instances
- Accidental exposure of admin ports
- Lateral movement — compromising one public service doesn't hand you the keys to the database

The bastion is a small, locked-down EC2 instance. VPN connection first, SSH second. Every admin action is effectively logged at the network level.

> *The castle has one gate. The bastion is the gate.*

---

## Why GitHub Actions → ECR → ECS/EC2?

**Reproducibility and auditability.**

A deploy that starts with `git push` and ends with a running container means:
- Every running version is a known, immutable image in ECR
- Rollbacks are image swaps, not `git revert` + manual deploys
- The deploy process is code — reviewable, versioned, reproducible

The pipeline enforces that **nothing runs in production that wasn't built from source control**. No hotfixes via SSH. No "I'll fix it directly on the server."

> *The pipeline is the contract between code and production.*

---

## Tradeoff Summary

| Decision | What we gain | What we give up |
|---|---|---|
| ASG over single server | Fault tolerance, scale | Operational complexity |
| Async SQS processing | Throughput, resilience | Immediate feedback to user |
| DocumentDB replica | Read scalability, isolation | Replication lag (eventual consistency on reads) |
| Dedicated AI Worker | Independent scaling, no API bleed | Another instance to manage |
| Redis for status | Sub-ms polling, zero DB load | One more stateful dependency |
| S3 for documents | Scalable, cheap, direct delivery | Slightly more complex download flow |
| Bastion-only SSH | Dramatically reduced attack surface | Extra VPN step for engineers |

---

*Architecture is a series of bets. Each decision above is a bet that the thing we gain is worth more than the thing we give up — given the specific constraints of an AI SaaS product where generation is slow, reliability is expected, and the team is small.*
