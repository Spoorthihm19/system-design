# System Design Interview: Delivery Framework

> The easiest way to fail a system design interview is to **not deliver a working system**. At mid-level this shows up as "time management", but usually the fix isn't working faster, it's **focusing on the right things**. A fixed structure keeps you from getting stuck and ensures you finish.

## Overview

| # | Step | Time | Primary goal |
|---|------|------|--------------|
| 1 | Requirements | ~5 min | Clarify the system (functional + non-functional) |
| 2 | Core Entities | ~2 min | Define the nouns/data of the system |
| 3 | API / System Interface | ~5 min | Define the contract with users |
| 4 | Data Flow *(optional)* | ~5 min | Describe processing steps (data-heavy systems) |
| 5 | High-Level Design | ~10-15 min | Satisfy **functional** requirements |
| 6 | Deep Dives | ~10 min | Satisfy **non-functional** requirements |

```
Requirements → Core Entities → API/Interface → (Data Flow) → High-Level Design → Deep Dives
              (Functional → HLD)                              (Non-functional → Deep Dives)
```

---

## 1. Requirements (~5 min)

Goal: get a clear understanding of what you're building. Split into two parts.

### Functional Requirements
"Users/Clients should be able to..." statements: the core features. Discuss them first, as a back-and-forth with the interviewer (ask like you'd ask a client/PM: *"Does the system need to do X?"*, *"What happens if Y?"*) to reach a **prioritized list**.

- **Twitter:** post tweets · follow users · see tweets from followed users
- **Cache:** insert items · set expirations · read items

**Keep it targeted: identify the top ~3.** Many systems have hundreds of features; a long list hurts you, and top companies evaluate your ability to prioritize.

### Non-Functional Requirements
System qualities that matter to users: "The system should be..." Put them **in context and quantify** where possible.

- Weak: "The system should be low latency."
- Strong: "The system should have low-latency search, < 500ms."

Twitter example:
- Highly available, prioritizing availability over consistency
- Scales to 100M+ DAU
- Low latency: feeds render in < 200ms

**Checklist (pick the top 3-5 most relevant):**

| Consideration | Questions to ask |
|---|---|
| **CAP theorem** | Consistency or availability? (Partition tolerance is a given.) |
| **Environment constraints** | Mobile battery, limited memory, limited bandwidth (e.g. streaming on 3G)? |
| **Scalability** | Bursty traffic, holiday/event spikes? Read vs. write ratio: which must scale more? |
| **Latency** | Which requests need meaningful computation (e.g. search in Yelp)? |
| **Durability** | Can data be lost? (social network: maybe; bank: never) |
| **Security** | Data protection, access control, regulations |
| **Fault tolerance** | Redundancy, failover, recovery |
| **Compliance** | Legal/regulatory/industry standards |

### Capacity Estimation (usually skip)
- Don't do upfront DAU/QPS/storage math just to conclude "it's a lot". It only shows you can do arithmetic.
- Tell the interviewer you'll **do math during the design, only when it affects a decision**.
- Example where it matters: **Top-K trending topics** → estimate number of topics to decide if a single min-heap suffices or it must be sharded.
- Still practice quick estimation; it helps reason about trade-offs. Don't worry if mental math is hard under pressure.

---

## 2. Core Entities (~2 min)

List the entities your API will exchange and your system will persist. A bulleted first draft is enough.

- **Why not the full data model now?** You don't know what you don't know. Start small, then add entities/fields as the design reveals them.
- **Twitter:** `User`, `Tweet`, `Follow`

Helpful questions:
- Who are the actors in the system? Do they overlap?
- What nouns/resources are needed to satisfy the functional requirements?

Choose good names; some interviewers use this to see if you can handle "one of the hardest problems in computer science".

---

## 3. API / System Interface (~5 min)

Define the contract between the system and its users. Usually maps directly to the functional requirements (not always).

### Protocol choice
| Protocol | Use when |
|---|---|
| **REST** | **Default.** HTTP verbs (GET/POST/PUT/DELETE) for CRUD on resources |
| **GraphQL** | Diverse clients with different data needs (avoids over/under-fetching) |
| **RPC (e.g. gRPC)** | Internal service-to-service calls where performance is critical |

Don't overthink it: default to REST. For real-time features add **WebSockets** or **Server-Sent Events**, but design the core API first.

### Twitter example (REST)
Resources are **plural nouns** based on core entities.

```http
POST /v1/tweets
body: { "text": string }

GET  /v1/tweets/{tweetId}  -> Tweet

POST /v1/follows
body: { "followee_id": string }

GET  /v1/feed              -> Tweet[]
```

> **Security rule:** derive the current user from the **auth token** in the request header, never from request bodies or path params. Never trust user-supplied IDs for identity.

---

## 4. Data Flow (~5 min, optional)

For backend/data-processing systems, list the high-level sequence of actions from input to output. **Skip if the system has no long sequence of actions.** Use it to inform the HLD.

Web crawler example:
1. Fetch seed URLs
2. Parse HTML
3. Extract URLs
4. Store data
5. Repeat

---

## 5. High-Level Design (~10-15 min)

Draw boxes and arrows (servers, databases, caches, etc.) showing how components interact. Use a whiteboard or tool like Excalidraw. **Ask your recruiter what tool is used and practice with it beforehand.**

**Principles**
- **Don't overthink.** Design an architecture that satisfies your API and therefore your requirements. Go **endpoint by endpoint**, building the design up sequentially.
- **Don't add complexity too early**: the most common reason candidates never reach a complete solution. Build a simple design for the core functional requirements, then layer complexity in deep dives.
- Spot a place for a cache/queue? Make a **verbal callout + written note**, then move on.
- **Talk through your thinking**: explain how data flows and what state (DB, cache, queue) changes on each request, from API request to response.
- When a request reaches the DB, **document the relevant columns/fields** next to it. Skip obvious ones (e.g. a User's name/email/password hash). Types aren't needed.

### Twitter example, built one endpoint at a time

1. `POST /v1/tweets`: Client → API Gateway → **Tweet CRUD Service** → **Tweet DB (Postgres)**
   - `Tweet`: `id`, `authorId`, `text`, `media: s3Links[]`
2. `GET /v1/tweets/:tweetId`: same path, read from Tweet DB
3. `POST /v1/follows`: API Gateway → **Follower Service** → **Following DB (Postgres)**
   - `Follow`: `userId1`, `userId2`
4. `GET /v1/feed`: API Gateway → **Feed Service** → `getFollowers()` (Follower Service) → `getTweets(followers)` (Tweet CRUD Service)

```
Client → API Gateway ─┬→ Tweet CRUD Service ──→ Tweet DB (Postgres)
                      ├→ Follower Service ────→ Following DB (Postgres)
                      └→ Feed Service ──┬─ getFollowers()
                                        └─ getTweets(followers)
```

---

## 6. Deep Dives (~10 min)

The simple Twitter HLD is "woefully inefficient" for feed fetching. That's exactly what deep dives are for. Use the remaining time to **harden the design**:

1. Ensure it meets all **non-functional requirements**
2. Address **edge cases**
3. Identify and fix **issues and bottlenecks**
4. Improve based on **interviewer probes**

**Twitter examples**
- **100M+ DAU** → horizontal scaling, caches, database sharding
- **Low-latency feeds** (the most interesting problem) → **fan-out on read vs. fan-out on write**, caching

**Seniority matters:** juniors can expect the interviewer to point out improvements; seniors should find the weak spots and **lead** the discussion.

**Common mistake: talking over the interviewer.** Be proactive but leave room for questions and probes; they likely have specific signals they want. Over-talking hurts your communication and collaboration evaluation.

---

## Quick Checklist

- [ ] Top ~3 functional requirements, prioritized
- [ ] 3-5 quantified non-functional requirements
- [ ] Skip upfront estimation; calculate only when it affects design
- [ ] Short list of core entities (iterate later)
- [ ] REST API (default), auth-derived user identity
- [ ] Simple HLD built endpoint by endpoint; note optimizations, don't build them yet
- [ ] Deep dives on scale, latency, edge cases, bottlenecks
- [ ] Leave room for interviewer questions
