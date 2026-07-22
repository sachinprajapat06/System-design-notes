# Weekend Practice — Week 1
### Week 1 | System Design | Sachin's Study Notes

---

## Goal of This Weekend

Don't read new material. **Consolidate what you learned.**

The best way to know if you understood something is to:
1. Explain it out loud (to yourself, to a friend, to a rubber duck)
2. Apply it to a blank page design problem
3. See where you get stuck — that's exactly where to re-read

Give yourself **Saturday for exercises, Sunday for review + self-test.**

---

## Exercise 1: CAP Decision Log for Your Real Systems

Open a blank document and fill this table from memory. Don't look at your notes until after.

| System You Used | CP or AP? | Why That Choice Made Sense | What Would Break If You Chose The Other? |
|----------------|-----------|---------------------------|------------------------------------------|
| PostgreSQL (BluePi ICICI) | | | |
| MongoDB (RXBenefits) | | | |
| DynamoDB (AWS Lambda) | | | |
| Redis at Cisco (sessions) | | | |
| Redis at Cisco (rate limits) | | | |
| SQS message queue | | | |

**Then check your Day 2 notes.** Where were you right? Where were you off?

**Reflection question:** Were there any systems where you were using AP but should have been using CP? (Hint: healthcare data, token revocation.)

---

## Exercise 2: Estimate Your SAP BTP Terraform Provider

This is a real system you work on. Estimate it properly.

### Assumptions to make (write them down as you assume them)

You know:
- SAP BTP has enterprise customers globally
- Terraform providers are used during CI/CD pipelines
- `terraform apply` calls BTP APIs and stores/reads state

Make a reasonable estimate for each:
- Number of SAP BTP customers running Terraform: ___
- Average number of Terraform workspaces per customer: ___
- Average `terraform apply` runs per workspace per day: ___
- Average number of API calls per `terraform apply`: ___
- Average state file size: ___

### Questions to answer

```
1. What is the total QPS to BTP APIs from Terraform?
   Formula: customers × workspaces × applies/day × API calls/apply ÷ 86400

2. What's the peak QPS? (Hint: when do CI/CD pipelines run?)

3. How much state storage is needed?
   Formula: customers × workspaces × state_file_size × version_history

4. What's the read:write ratio?
   (reads = terraform plan, writes = terraform apply)

5. What does this tell you about the architecture?
   - Is this a high-QPS problem or a correctness problem?
   - What's the most important non-functional requirement?
```

**Sample numbers to validate against (fill in yours first, then compare):**
```
Rough estimate:
  50,000 SAP customers × 10 workspaces × 5 applies/day × 50 API calls
  = 25,000,000 API calls/day
  = 289 API calls/second avg
  Peak (9 AM - 11 AM business hours): ~1,000 QPS

  State storage:
  50,000 × 10 × 100 KB × 30 versions = 1.5 TB

  Read:Write = maybe 3:1 (terraform plan >> terraform apply)

  Conclusion: This is moderate QPS with strong consistency requirements.
  S3 for state storage + DynamoDB for locking = correct architecture.
  NOT a massive scale problem.
```

---

## Exercise 3: Draw a URL Shortener — Blank Page, No Notes

**Time yourself: 30 minutes maximum.**

**The question:** Design a URL shortener service (like bit.ly).

Requirements:
- 100 million URLs stored
- 10 billion redirects per month
- Custom aliases supported
- Redirect latency must be < 10ms

### Step 1: Estimation (5 min)

Fill these in:

```
Redirect QPS (avg): 10B ÷ 86,400 ÷ 30 = _____ /sec
Redirect QPS (peak): _____ × 3 = _____ /sec

Write QPS (100M URLs, assume spread over 1 year):
  100M ÷ 365 ÷ 86,400 = _____ /sec

Storage:
  Each URL: ~500 bytes (short code + long URL + metadata)
  100M × 500 bytes = _____ GB

Read:Write ratio: _____ : 1
```

### Step 2: Draw the Architecture (15 min)

Draw boxes and arrows for:
- Client (browser)
- Write path: creating a short URL
- Read path: redirecting a short URL
- Data storage layer
- Cache layer

Label each arrow with what's being sent.

### Step 3: Deep Dive on 2 Questions (10 min)

Answer in writing:

**Question A: How do you generate the short code?**

Think about:
- What approaches exist?
- What are the trade-offs?
- Which would you pick and why?

**Question B: How do you achieve < 10ms redirect latency?**

Think about:
- What's the latency of a PostgreSQL lookup? Fast enough?
- What's the latency of a Redis lookup?
- At 4,000 QPS, how many Redis instances do you need?

---

### Expected Answer to Check Yourself Against

```
Estimation:
  Redirect QPS avg:  10B ÷ (30 × 86400) ≈ 3,858/sec ≈ 4,000/sec
  Redirect QPS peak: 4,000 × 3 = 12,000/sec
  Write QPS: 100M ÷ (365 × 86400) ≈ 3.2/sec (very low!)
  Storage: 100M × 500B = 50 GB
  Read:Write: 4,000 : 3 = 1,333 : 1 (extremely read heavy)

Architecture:
  ┌──────────────┐           ┌───────────────────┐
  │   Browser    │           │   Write Service   │
  │              │──POST──▶  │  (3 QPS — tiny)   │
  │ GET /aB3kR9 │           │  Generate code    │
  └──────┬───────┘           │  Save to DB + cache│
         │                   └─────────┬─────────┘
         ▼                             │
  ┌──────────────┐                     ▼
  │ Load Balancer│           ┌───────────────────┐
  └──────┬───────┘           │   PostgreSQL DB   │
         │                   │  (source of truth) │
         ▼                   └───────────────────┘
  ┌──────────────┐                     ▲
  │ Redirect     │           cache miss│
  │ Service      │──────▶  ┌───────────────────┐
  │              │         │   Redis Cache     │
  └──────────────┘         │  (50GB — fits!)   │
       │ cache hit         └───────────────────┘
       ▼
  HTTP 301 → long URL

Code Generation:
  Option A: hash(long_url) → base62 → take first 7 chars
    Problem: collision possible
    Solution: append counter if collision detected
    
  Option B: auto-increment ID → base62 encode
    No collisions. id=1 → "a", id=2 → "b", ..., id=3.5T → "aaaaaaa"
    Problem: sequential IDs are guessable/enumerable
    Solution: start from random offset + shuffle the base62 alphabet
    
  Pick B with random offset. Simple, collision-free, fast.

Latency < 10ms:
  DB lookup for a 50GB dataset: ~10ms (borderline!)
  Redis lookup: ~1ms ✓
  
  With 4,000 QPS of redirects:
    Cache hit rate target: 99% (most URLs are accessed repeatedly)
    Redis QPS: 4,000 (Redis handles 100K+ QPS easily — 1 instance is fine)
    DB QPS from cache misses: 4,000 × 1% = 40 QPS (trivial for PostgreSQL)
```

---

## Exercise 4: Availability Budget Math

Your team's API has a 99.9% SLA.

**Scenario:**
- January: 1 server crash → 45 minutes downtime
- February: Planned deployment (rolling deploy not set up) → 20 minutes downtime
- March: A bug caused a cascade failure → 3 hours downtime

**Questions:**

1. How many total minutes of downtime are allowed per year at 99.9%?
2. How many minutes have you used by end of March?
3. How many minutes remain for April–December?
4. You have a big feature deploy in April that needs 30 minutes downtime. Can you "afford" it?
5. What two things would you fix first to protect your remaining availability budget?

```
Answers:
1. 99.9% allows: 365 × 24 × 60 × 0.001 = 525.6 minutes/year
   ≈ 8 hours 46 minutes per year

2. Used: 45 + 20 + 180 = 245 minutes

3. Remaining: 525.6 - 245 = 280.6 minutes for next 9 months

4. 30 minutes for the deploy: yes, technically you can "afford" it.
   But then you have 250 minutes left for 8 months = 31 min/month.
   Any significant incident in those 8 months = SLA breach.
   You should set up zero-downtime deployment BEFORE this deploy.

5. Fix 1: Zero-downtime deployment (rolling deploy)
     Eliminates 20+ min/deploy × however many deploys you do
   Fix 2: Circuit breakers + better error handling
     Prevents cascade failures (that 3-hour outage was likely avoidable)
```

---

## Exercise 5: The Latency Chain Calculation

Your new feature at SAP BTP makes these service calls **sequentially**:

1. Auth Service: validate JWT → 20ms (Redis lookup)
2. BTP Tenant Service: get tenant info → 50ms (PostgreSQL query)
3. Terraform State Service: read state file → 80ms (S3 read)
4. Policy Service: check RBAC permissions → 30ms (Redis cache)

**Questions:**

1. What's the total latency if these calls are sequential (one after another)?
2. What's the total latency if calls 1 and 4 can run in parallel? (They don't depend on each other)
3. What's the latency if calls 2 and 3 can also run in parallel with each other (after auth)?
4. What's the theoretical minimum latency if you can run as much in parallel as possible?

```
Answers:
1. Sequential: 20 + 50 + 80 + 30 = 180ms

2. Auth (20ms) runs, then parallel [Tenant (50ms), State (80ms), Policy (30ms)]
   Wait, Policy doesn't depend on auth output? Let's say it doesn't.
   Auth + Policy in parallel: max(20, 30) = 30ms
   Then Tenant + State in parallel: max(50, 80) = 80ms
   Total: 30 + 80 = 110ms (vs 180ms sequential — 39% faster)

3. Auth parallel with Policy: 30ms
   Then Tenant parallel with State: 80ms  
   Total: 110ms (same as above)

4. If we can run all 4 in parallel: max(20, 50, 80, 30) = 80ms
   But this only works if none depend on each other's output.
   In practice, auth must complete before you know WHO is making the request.
   So: auth (20ms) → then rest in parallel (80ms) = 100ms
```

**Key lesson:** Go's goroutines make parallelizing these calls trivial. A well-designed Go service does this naturally with `sync.WaitGroup` or `errgroup`. This is a Team Lead question: "Can we parallelize calls 2 and 4?"

---

## Sunday Self-Test — Can You Explain These?

If you can explain each concept out loud in 60 seconds without notes, you've learned it.

Try each one. Say it to yourself or write it down.

```
1. Explain horizontal scaling to someone who has never coded.
   (Use an analogy. Don't say "stateless" — say what it means.)

2. Explain the CAP theorem.
   Give one example of a system that should be CP and why.
   Give one example of a system that should be AP and why.

3. What does "99.9% availability" actually mean in hours per year?
   What's the first thing you'd fix to improve from 99.9% to 99.99%?

4. If 500M users use an app, each making 10 requests/day,
   what's the average QPS? What's a reasonable peak QPS estimate?

5. What's the difference between L4 and L7 load balancers?
   When would you use each one?

6. What is TTL in DNS and why does it matter for deployments?

7. What's the difference between:
   - An SLA (Service Level Agreement)
   - An SLO (Service Level Objective)
   - An SLI (Service Level Indicator)
```

---

## Week 1 Completion Checklist

Mark each box when you're confident:

```
[ ] I can explain vertical vs horizontal scaling with an analogy
[ ] I can name the CAP theorem properties and what each means
[ ] I can look at a system (PostgreSQL, DynamoDB, Redis) and say CP or AP
[ ] I know that 99.9% = 8.76 hours downtime/year
[ ] I know the 4 patterns for high availability
[ ] I can convert "100M events/day" to QPS in my head (≈ 1,200/sec)
[ ] I know: Redis ~1ms, DB ~10ms, S3 ~100ms
[ ] I can explain what DNS does and why TTL matters
[ ] I know the difference between L4 and L7 load balancers
[ ] I know what CDN is for and what "cache-busting" means
```

**If you have 8+ checks: move on to Week 2 confidently.**  
**If you have fewer: identify which concepts need more time and re-read those days.**

---

## Preview: What Week 2 Will Cover

Next week we go deep on **databases** — the component that is almost always the bottleneck in real systems.

You'll learn:
- Why SQL and NoSQL are not "better/worse" but suited to different workloads
- How indexes actually work (why `WHERE email = ?` is fast, `WHERE email LIKE '%@sap.com'` is slow)
- Replication: how databases copy data across multiple servers
- Sharding: how to split a database when it's too big for one machine
- Schema design patterns used in production (like your ICICI Lombard work)

*See you in [Week 2](../week-2/)*

---

*Back to [Week 1 Overview](../week1_scalability_estimation.md)*
