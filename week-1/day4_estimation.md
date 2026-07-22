# Day 4 — Back-of-Envelope Estimation
### Week 1 | System Design | Sachin's Study Notes

---

## What Are We Learning Today?

Estimation is not about being precise. It's about being **right by an order of magnitude**.

The goal: when someone asks "how many servers do we need?" or "how much will S3 cost us?" you should be able to answer in 2 minutes — not with a spreadsheet, but with fast mental math.

This is a **Team Lead superpower.** In design reviews, you can immediately challenge assumptions. In interviews, it shows senior-level thinking. In project planning, it prevents you from designing a system that's 100x too small or 100x too expensive.

**The motto:** "Roughly right is better than precisely wrong."

---

## The Building Blocks — Numbers You Must Memorize

Don't just read these. Write them down. Internalize them. They're your toolkit.

### Time Conversions (Most Useful)

```
60 seconds       = 1 minute
3,600 seconds    = 1 hour
86,400 seconds   = 1 day  ← CRITICAL — memorize this
604,800 seconds  = 1 week
2,592,000 seconds ≈ 1 month (30 days)
31,536,000 seconds = 1 year

Quick approximations:
  1 million / 86,400 ≈ 11.5 → round to 12
  So: 1 million events/day ≈ 12 events/second (QPS)
  
  1 billion / 86,400 ≈ 11,574 → round to 12,000
  So: 1 billion events/day ≈ 12,000 events/second (12K QPS)
```

**Shortcut table:**

| Daily Events | Approx QPS |
|-------------|------------|
| 1 million | ~12/sec |
| 10 million | ~120/sec |
| 100 million | ~1,200/sec (1.2K) |
| 1 billion | ~12,000/sec (12K) |
| 10 billion | ~120,000/sec (120K) |

### Storage Units (Build the Ladder)

```
1 Byte     = 8 bits
1 KB       = 1,000 bytes (kilobyte)
1 MB       = 1,000 KB   = 1,000,000 bytes
1 GB       = 1,000 MB   = 1,000,000,000 bytes = 10^9
1 TB       = 1,000 GB   = 10^12 bytes
1 PB       = 1,000 TB   = 10^15 bytes

Memory shortcut:
  ASCII character = 1 byte
  Unicode character (UTF-8) = 1-4 bytes (avg ~2)
  Average English word = 5 chars ≈ 5 bytes
  Tweet (280 chars) ≈ 300 bytes with metadata
  Photo (compressed JPEG, phone) ≈ 3-5 MB
  HD video (1 minute, compressed) ≈ 100 MB
  
Quick conversions:
  1 million × 1 KB  = 1 GB
  1 billion × 1 KB  = 1 TB
  1 million × 1 MB  = 1 TB
  1 billion × 1 MB  = 1 PB
```

### Latency Numbers Every Engineer Should Know

These were famously compiled by Jeff Dean at Google. They show how fast different operations are:

```
Operation                    Latency      Notes
─────────────────────────────────────────────────────────────────
L1 cache reference           0.5 ns       
Branch mispredict            5 ns         
L2 cache reference           7 ns         14x L1 cache
Mutex lock/unlock            25 ns        
Main memory (RAM) reference  100 ns       200x L1 cache
Compress 1 KB (Snappy)       3,000 ns     3 µs
Send 1 KB over network       10,000 ns    10 µs
Read 4 KB from SSD           150,000 ns   150 µs (SSD fast!)
Read 1 MB from RAM           250,000 ns   250 µs
Round-trip in datacenter     500,000 ns   0.5 ms
Read 1 MB from SSD           1,000,000 ns 1 ms
HDD seek + read              10,000,000 ns 10 ms (HDD slow!)
Send packet CA → Netherlands  150,000,000 ns 150 ms
─────────────────────────────────────────────────────────────────
```

**What to remember (simplified):**
```
Redis lookup:          ~1 ms   (in-memory, same datacenter)
PostgreSQL query:      ~10 ms  (SSD, simple indexed query)
S3/external API call:  ~100 ms (network, external service)
Cross-continent:       ~150 ms (physics limit of light)
```

**Why this matters:**
- Redis is 10x faster than a DB query
- A DB query is 10x faster than an S3 call
- If your API calls 5 services serially: 5 × 100ms = 500ms latency
- Use this to detect obvious performance issues in design reviews

---

## The Estimation Template — Step by Step

Every estimation follows this structure. Learn it like a recipe.

```
Step 1: Users
  - Daily Active Users (DAU)
  - Monthly Active Users (MAU)
  - Growth rate

Step 2: Traffic (QPS)
  - Avg requests per user per day
  - Total QPS = (DAU × requests) / 86,400
  - Peak QPS = avg × 2 to 3 (traffic isn't uniform — spikes happen)
  - Read/write ratio (usually reads >> writes)

Step 3: Storage
  - New data per day = writes/day × avg object size
  - Total storage = new data per day × retention period

Step 4: Bandwidth
  - Inbound = write QPS × avg request size
  - Outbound = read QPS × avg response size

Step 5: Servers (if needed)
  - Assume 1 server handles ~1,000 QPS (simple rule of thumb)
  - Servers = Peak QPS / 1,000
```

---

## Full Worked Example 1: Instagram-Like Photo App

Let's estimate the storage and bandwidth for a photo sharing service.

### Given (make reasonable assumptions and state them)
- 500 million MAU (Monthly Active Users)
- 100 million DAU
- Average user uploads 2 photos/day
- Average user views 30 photos/day
- Average photo size: 5 MB (original), 200 KB (compressed thumbnail)

### Step 1: Traffic

```
Writes (uploads):
  100M users × 2 photos/day = 200M photo uploads/day
  200M / 86,400 ≈ 2,300 uploads/second (2.3K write QPS)
  Peak: ~7,000 uploads/second (evening peak)

Reads (views):
  100M users × 30 views/day = 3 billion photo views/day
  3B / 86,400 ≈ 35,000 reads/second (35K read QPS)
  Peak: ~100,000 reads/second (10x the write QPS)
  
  Read:Write ratio = 35,000 : 2,300 ≈ 15:1
  → Read-heavy system → caching is critical
```

### Step 2: Storage

```
Photos per day:    200M photos
Average size:      5 MB (original) + 200 KB (thumbnail)
                 ≈ 5.2 MB per photo (let's round to 5 MB)

Storage per day:   200M × 5 MB = 1,000,000 MB = 1,000 GB = 1 TB/day

After 10 years:    1 TB/day × 365 × 10 = 3,650 TB = 3.65 PB

Reality check: Instagram actually stores ~100 PB+ 
because they store multiple sizes and formats. Our estimate is in the right ballpark.
```

### Step 3: Bandwidth

```
Upload bandwidth:
  2,300 uploads/sec × 5 MB = 11,500 MB/sec = 11.5 GB/sec
  → Massive inbound bandwidth — needs distributed upload servers

Download bandwidth:
  35,000 reads/sec × 200 KB (thumbnail) = 7,000,000 KB/sec = 7 GB/sec
  → CDN is essential — 7 GB/sec of global traffic from a single DC would be insane
```

### Step 4: Servers

```
Read servers:
  35K QPS ÷ 1,000 QPS per server = 35 read servers minimum
  With peak: 100K ÷ 1,000 = 100 read servers

Write/upload servers:
  2.3K QPS ÷ 1,000 = ~3 upload servers (separate from read)

But these are MINIMUMS — in production, you add 2x for redundancy and failover.
Total: ~200 app servers for the whole system.
```

---

## Full Worked Example 2: Cisco IAM System (Your Past Work)

Let's estimate what a real IAM system needs.

### Given Assumptions
- Cisco: ~30,000 employees, ~200 internal apps
- Each employee authenticates ~5 times/day
- Each app checks user permissions ~100 times per session
- Sessions last ~8 hours
- Token TTL: 15 minutes (must refresh frequently)

### Traffic

```
Auth requests (logins):
  30,000 employees × 5 logins/day = 150,000 logins/day
  150,000 / 86,400 ≈ 1.7 logins/second
  Peak (9 AM Monday morning): ~50 logins/second (everyone logs in at once)

Permission checks (every API call checks auth):
  30,000 employees × 8 hours/day × 100 permission checks/hour
  = 30,000 × 800 = 24,000,000 checks/day
  24M / 86,400 ≈ 278 checks/second
  Peak (9-10 AM): ~1,000 checks/second

Token refresh requests:
  Each session = 8 hours = 480 minutes
  With 15-min TTL: each session refreshes 480/15 = 32 times
  30,000 employees × 32 refreshes/day = 960,000 refreshes/day
  ≈ 11 refreshes/second
```

### The Key Insight

```
Total IAM QPS:
  Logins:             50/sec (peak)
  Permission checks: 1,000/sec (peak) ← DOMINANT workload
  Token refresh:       11/sec
  
  Total: ~1,061 QPS peak
  
  With 1 server at 1,000 QPS: you need 2 servers for peak.
  With 2x redundancy: 4 servers total.
  
  But permission checks are the bottleneck.
  → Cache permission results in Redis
  → Reduce DB lookups from 1,000/sec to ~10/sec (cache hit rate 99%)
  → Much smaller DB footprint needed
```

### Storage

```
Active sessions:
  30,000 employees, avg 2 concurrent sessions per person
  = 60,000 sessions
  Each session: 2 KB (user ID, token, claims, expiry)
  Total: 60,000 × 2 KB = 120 MB → fits easily in Redis!
  
User/role/permission data:
  30,000 users × 1 KB per user record = 30 MB
  Roles: 100 roles × 1 KB = 100 KB
  Permissions: 1,000 permissions × 100 bytes = 100 KB
  
  Total: ~30 MB → PostgreSQL handles this effortlessly
  
Audit logs:
  ~1M auth events/day × 500 bytes = 500 MB/day
  Retain 1 year: 500 MB × 365 = 182 GB
  → S3 + Athena for cheap long-term audit storage
```

---

## Full Worked Example 3: ICICI Lombard Insurance (Your BluePi Work)

### Given Assumptions
- 50 million policyholders
- 500,000 DAU (1% active on any day)
- 3 reads per user (check policy, check claim, download document)
- 100,000 new claims per day
- Average policy document: 500 KB

### Traffic

```
Reads:
  500,000 users × 3 reads = 1,500,000 reads/day
  1.5M / 86,400 ≈ 17 reads/second
  Peak (lunch hours): ~50 reads/second
  
  → Very low QPS. A single PostgreSQL handles this with no problem.

Claim submissions:
  100,000/day / 86,400 ≈ 1.15 claims/second
  
  → Also very low. This is NOT a high-QPS system.
  → The design challenge here is DATA INTEGRITY, not scale.
```

### Storage

```
Policy documents:
  50M policyholders × 500 KB = 25 TB
  But not all policies are active — say 20M active
  20M × 500 KB = 10 TB of active policy data
  
  → Store in S3 ($0.023/GB/month = $0.023 × 10,000 = $230/month for 10 TB)
  → Very cheap! S3 is the right choice for documents.

Claim data:
  100,000 claims/day
  Retain for 7 years (regulation)
  100,000 × 365 × 7 × 10 KB per claim = 2.55 TB
  → PostgreSQL can handle this, possibly with partitioning by year
  
Database rows:
  50M policy rows → large but manageable for PostgreSQL
  At 1 KB per row: 50 GB of policy data
  With indexes: ~100 GB
  → Single PostgreSQL instance with good SSDs handles this fine
```

### Key insight from this estimation

```
ICICI Lombard is NOT a scaling problem — it's a data integrity problem.
  - 50 reads/second is trivially easy for a DB
  - The challenge is: wrong data in insurance = legal liability
  
This is why you correctly used PostgreSQL (CP, ACID) here.
The CAP theorem decision aligns with the estimation result.

A junior engineer might over-engineer this:
  "50M users! We need Kafka and DynamoDB!"
  
You now know: 50 reads/second — 1 PostgreSQL server handles this easily.
  The architecture question is: how do we ensure data correctness?
  Not: how do we handle massive scale?
```

---

## The Common Estimation Mistakes

### Mistake 1: Forgetting to separate read from write

```
"We have 1,000 QPS"
→ Is that reads, writes, or both?

Reads and writes have different bottlenecks:
  - Reads: solved by caching, read replicas
  - Writes: harder — must hit primary DB, can't easily cache
  
Always split QPS into read QPS + write QPS and handle them separately.
```

### Mistake 2: Not accounting for peak traffic

```
"Average QPS is 100, so we need 1 server"

But traffic is NOT uniform:
  - 3 AM: 10 QPS
  - 9 AM Monday: 400 QPS (everyone starts work)
  - 12 PM Friday: 300 QPS
  
Design for peak, not average. Use peak = 2-3x average as a starting point.
For product launches or flash sales: 10x+ spike is possible.
```

### Mistake 3: Ignoring growth

```
"Today we have 10,000 users, so we need 1 server"

But if you're growing 50% per year:
  Year 1: 10,000 users
  Year 2: 15,000 users
  Year 3: 22,500 users
  
Design today's architecture to handle 3 years of growth without a complete rewrite.
Otherwise you'll rebuild the system every year.
```

### Mistake 4: Estimating wrong unit of data

```
"A user profile is tiny — maybe 100 bytes"

But wait:
  - Profile photo URL: 100 chars = 100 bytes
  - Name: 50 bytes
  - Email: 50 bytes  
  - Bio: 500 bytes
  - Metadata: 200 bytes
  - Row overhead in DB: ~100 bytes
  
  Total: ~1 KB per user (not 100 bytes)
  
  At 50M users: 1 KB × 50M = 50 GB (not 5 GB)
  Big difference in hardware planning.
```

---

## Speed Run Estimation Practice

Try these yourself in 2 minutes each before reading the answers:

### Problem 1: WhatsApp Message Storage
- 2 billion users
- 100 messages/user/day
- Average message: 100 bytes
- Retain messages: 5 years
- How much storage?

```
Messages/day: 2B × 100 = 200 billion messages/day
Storage/day: 200B × 100 bytes = 20,000 GB = 20 TB/day
5 years: 20 TB × 365 × 5 = 36,500 TB = 36.5 PB

With media (voice notes, photos, videos):
  Avg message size is more like 5 KB (mix of text + media)
  5 years: 20 TB × 50 (5 KB instead of 100 bytes) = 1.8 EB

Reality check: WhatsApp processes ~100 billion messages/day.
Our estimate of 200B is slightly high but in the right order of magnitude.
```

### Problem 2: SAP BTP Terraform State Storage
- 50,000 SAP enterprise customers
- Each customer has 10 Terraform workspaces
- Each state file: 100 KB
- State updated 10 times/day
- Retain 30 days of history

```
Total workspaces: 50,000 × 10 = 500,000 workspaces
Total state files: 500,000 × 30 (history) = 15 million files
Total storage: 15M × 100 KB = 1,500,000 MB = 1,500 GB = 1.5 TB

State updates (writes):
  500,000 workspaces × 10 updates/day = 5M updates/day
  5M / 86,400 ≈ 58 writes/second (very manageable)

This is easily handled by S3 + DynamoDB for state locking.
Not a scale problem — a consistency/locking problem.
```

### Problem 3: Healthcare Portal (BluePi)
- 500,000 patients
- 1 doctor visit recorded per patient per 3 months = 4 per year
- Each medical record: 10 KB
- Retain: forever (legal requirement)
- 50,000 DAU checking their records

```
New records/year: 500,000 × 4 = 2M records/year
Storage growth/year: 2M × 10 KB = 20,000 MB = 20 GB/year

After 20 years: 20 GB × 20 = 400 GB
→ Tiny! PostgreSQL handles this on a single server forever.

Read traffic:
  50,000 DAU × 3 reads/day = 150,000 reads/day
  = ~1.7 reads/second
→ Trivially low QPS. Even a single DB handles this easily.

Again: NOT a scale problem. It's a compliance + data integrity problem.
→ This validates your PostgreSQL choice at BluePi.
```

---

## Summary — 5 Things to Remember

```
1. 86,400 seconds in a day. This converts "per day" to QPS.
   1M events/day ≈ 12 QPS. Memorize this.

2. Storage ladder: KB → MB → GB → TB → PB
   1M × 1KB = 1GB. 1B × 1KB = 1TB. Use these as building blocks.

3. Always split Read QPS and Write QPS — they're solved differently.
   Reads: cache them. Writes: much harder to scale.

4. Design for peak (2-3x average), not average traffic.
   And design for 3 years of growth, not just today.

5. Sometimes the estimation TELLS you the architecture.
   If QPS is < 100, your problem is NOT scale — it's correctness or features.
   If QPS > 100K, your problem IS scale — you need caching and horizontal scaling.
```

---

## Quick Quiz

1. A news app has 50 million DAU. Each user reads 20 articles/day and writes 0.1 comments/day. What's the read QPS and write QPS? What's the read:write ratio?

2. A user profile row is 2 KB. Your app has 100 million users. How much storage for user profiles?

3. Your CI/CD system runs 500 build jobs per day. Each build logs 50 MB of output. You retain logs for 90 days. How much log storage do you need?

*(Answers:
1. Reads: 50M × 20 / 86,400 ≈ 11,574 ≈ 12K QPS. Writes: 50M × 0.1 / 86,400 ≈ 58 QPS. Ratio: 12,000:58 ≈ 200:1 read-heavy.
2. 100M × 2 KB = 200,000 MB = 200 GB.
3. 500 × 50 MB = 25,000 MB = 25 GB/day. 90 days: 25 × 90 = 2,250 GB = 2.25 TB.)*

---

*Next: [Day 5 — DNS, Load Balancers & CDN Overview]
