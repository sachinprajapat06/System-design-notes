# Day 3 — Availability & SLAs
### Week 1 | System Design | Sachin's Study Notes

---

## What Are We Learning Today?

Every system goes down sometimes. The question is: **how often, and for how long?**

An SLA (Service Level Agreement) is a promise you make to your users about uptime. It sounds like a business term, but it directly drives **every architectural decision you make** — how many servers you run, whether you use redundancy, how you write your health checks, and whether you can afford planned maintenance.

Today you'll learn to think in "nines" (99%, 99.9%, 99.99%) and understand what they really mean in minutes of downtime per year.

---

## The "Nines" of Availability

Availability is expressed as a percentage of time the system is working.

| SLA | Downtime/Year | Downtime/Month | Downtime/Day | Real-World Use |
|-----|--------------|----------------|--------------|----------------|
| 90% | 36.5 days | 3 days | 2.4 hours | Nobody ships this intentionally |
| 99% | 3.65 days | 7.3 hours | 14.4 minutes | Internal tools only |
| 99.9% | 8.76 hours | 43.8 minutes | 1.44 minutes | Standard web services |
| 99.99% | 52.6 minutes | 4.4 minutes | 8.6 seconds | Payment systems, important APIs |
| 99.999% | 5.26 minutes | 26.3 seconds | 0.86 seconds | Telecom, banking core |
| 99.9999% | 31.5 seconds | 2.6 seconds | ~0 | Mission critical infra |

### Let That Sink In

**99.9% sounds excellent.** "We're down only 0.1% of the time!"
But 0.1% of a year = **8 hours and 46 minutes of downtime every year**.

For your IAM service at Cisco:
- 30+ teams depended on it
- If IAM is down, NOBODY can authenticate
- 8 hours 46 minutes of auth failure per year = very bad

**99.99% for a payment system:**
- 52.6 minutes per year
- If you process ₹10 crore per hour → 52 minutes down = ₹86 lakh of transactions lost
- Now you see why payment companies invest heavily in 99.99%

---

## How Availability Is Calculated

```
Availability = Uptime / (Uptime + Downtime)

Example: System was down for 4 hours in a 30-day month
  Uptime = 30 days × 24 hours - 4 hours = 716 hours
  Total  = 720 hours (30 days)
  
  Availability = 716 / 720 = 0.9944 = 99.44%
  
  Is that 99.9%? No — it falls short.
  SLAs are not "approximately" — they're contractual.
```

### Compounding Availability (The Hidden Problem)

When you chain multiple services together, the total availability MULTIPLIES.

```
Request path: Client → API Gateway → Auth Service → Business Service → Database

API Gateway:     99.99%
Auth Service:    99.9%
Business Service: 99.9%
Database:        99.9%

Total Availability = 0.9999 × 0.999 × 0.999 × 0.999
                   = 0.9969
                   = 99.69%

You aimed for 99.9% but you're only hitting 99.69%
because the components multiply, not add.
```

**This is a critical insight for system design.**
If you have 10 microservices chained together, each at 99.9%, your total is:
`0.999^10 = 0.99 = 99%` — only 2 nines!

**What this means for you:**
- At SAP BTP with microservices, the chain length matters
- Each additional service in the request path degrades total availability
- You need your critical services (like auth) at a HIGHER SLA than your target
- Or you use async patterns (SQS) to decouple the chain

---

## The Real Components of Availability

### 1. Mean Time Between Failures (MTBF)

How long does the system run before it fails?
```
If your server crashes once every 30 days:
  MTBF = 30 days = 720 hours
```

You increase MTBF by:
- Writing better code (fewer bugs)
- Better hardware
- Redundancy (if 1 fails, another takes over — effectively infinite MTBF from user's perspective)

### 2. Mean Time To Recovery (MTTR)

After something fails, how long until it's working again?
```
If your server crashes and it takes 20 minutes to detect + restart:
  MTTR = 20 minutes
```

You decrease MTTR by:
- Auto-healing (Kubernetes restarts pods automatically)
- Good monitoring and alerts (know about failures in seconds, not hours)
- Runbooks (documented steps to fix known issues)
- Blue-green deployments (instant rollback)

### The Formula
```
Availability = MTBF / (MTBF + MTTR)

MTBF = 720 hours, MTTR = 1 hour:
  Availability = 720 / (720 + 1) = 99.86% ≈ 99.9%

MTBF = 720 hours, MTTR = 0.1 hours (6 minutes):
  Availability = 720 / (720 + 0.1) = 99.986% ≈ 99.99%

Halving MTTR takes you from 99.9% to 99.99%!
→ Faster recovery = higher availability
→ This is why great monitoring is a high-SLA requirement, not a nice-to-have
```

---

## How to Achieve High Availability: The Patterns

### Pattern 1: Redundancy (No Single Point of Failure)

**The core rule:** For every critical component, have at least one backup ready to take over.

```
BEFORE (single point of failure):
  Client → [Single Load Balancer] → [Single App Server] → [Single DB]
  
  If ANY of those 3 goes down → entire system is down

AFTER (redundant):
  Client → [LB 1] [LB 2] → [App Server 1] [App Server 2] [App Server 3]
                                         → [DB Primary] [DB Replica]
  
  App Server 2 goes down → LB routes to App Server 1 and 3 → no user impact
  DB Primary crashes → Replica is promoted → brief interruption only
```

**Redundancy levels:**
- **Component-level:** 2 app servers, 2 load balancers
- **Zone-level:** Servers in 2 AWS Availability Zones (separate data centers in same region)
- **Region-level:** Servers in Mumbai AND Singapore (survives a whole region outage)

**Your AWS context:**
When you deploy Lambda + DynamoDB, AWS automatically spreads them across multiple AZs. That's why Lambda has a 99.95% SLA — redundancy is built in.

### Pattern 2: Health Checks + Auto-Restart

If a server goes down, detect it and bring it back — automatically.

```
Health Check Flow:
  Load Balancer → polls GET /health every 10 seconds
  
  Normal:
    /health returns 200 OK
    LB keeps sending traffic ✓
  
  Failure:
    /health returns 503 or times out
    LB immediately stops sending traffic to this server
    Kubernetes/ECS detects: pod is unhealthy
    Kubernetes restarts the pod
    /health returns 200 again
    LB resumes sending traffic
    
  Time for user: 10-20 seconds of that server's traffic shifted away
  User impact: possibly 1-2 failed requests, then back to normal
```

**A good `/health` endpoint in Go:**
```go
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    // Check things that matter:
    
    // 1. Can we reach the database?
    if err := db.PingContext(r.Context()); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "unhealthy",
            "reason": "database unreachable",
        })
        return
    }
    
    // 2. Can we reach Redis?
    if err := redisClient.Ping(r.Context()).Err(); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "unhealthy", 
            "reason": "cache unreachable",
        })
        return
    }
    
    // All good
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "healthy"})
})
```

### Pattern 3: Graceful Degradation

When a dependency fails, don't fail completely — serve a reduced but still-useful response.

**The concept:** Your system has "tiers" of functionality. When a non-critical tier fails, the core still works.

```
Example: Insurance policy portal (BluePi)
  
  Core feature: View your insurance policy ← MUST work
  Optional feature: Real-time premium calculator ← Nice to have
  Optional feature: Chat support widget ← Nice to have
  
  Graceful degradation:
    If real-time calculator service is down:
      → Show policy without calculator
      → "Premium estimator temporarily unavailable"
      → User can still VIEW their policy (core feature works)
    
    If chat service is down:
      → Hide chat widget
      → Show "Contact us at email@company.com"
      → User can still do everything except chat
    
    If DATABASE is down:
      → Check Redis cache first
      → Serve cached version of the policy (might be 5 min old)
      → Show "Data may not be current" banner
      → Better than showing a 500 error
```

```go
// Example in Go: graceful degradation with cache fallback
func GetPolicy(policyID string) (*Policy, error) {
    // Try cache first
    cached, err := cache.Get("policy:" + policyID)
    if err == nil {
        return cached, nil
    }
    
    // Try database
    policy, err := db.GetPolicy(policyID)
    if err != nil {
        // DB is down — if we have stale cache, serve it with a warning
        stale, cacheErr := cache.GetStale("policy:" + policyID)
        if cacheErr == nil {
            stale.Warning = "Data may be up to 10 minutes old"
            return stale, nil
        }
        // Nothing available — now we return an error
        return nil, fmt.Errorf("policy service temporarily unavailable")
    }
    
    // DB worked — update cache for next time
    cache.Set("policy:"+policyID, policy, 10*time.Minute)
    return policy, nil
}
```

### Pattern 4: Circuit Breaker

When a downstream service is failing, **stop hitting it** — let it recover.

**The analogy:** Your house's circuit breaker. When a wire short-circuits, the breaker OPENS and stops electricity flow. Why? Because keep sending electricity = fire. The breaker protects the rest of the house.

```
Without circuit breaker:
  Your service → calls Payment API → Payment API is down
  Each request waits 30 seconds for timeout
  Your service has 100 requests/sec
  After 30 sec: 3,000 pending requests all waiting
  Your service: out of threads/memory → YOUR service crashes
  → One failing dependency brought down your service

With circuit breaker:
  State: CLOSED (normal) — all requests go through
  
  10 requests in a row fail → State changes to OPEN
    All requests immediately return error (no waiting)
    Your service stays healthy
    
  After 30 seconds → State changes to HALF-OPEN
    1 test request goes through
    If it succeeds → State: CLOSED (back to normal)
    If it fails → State: OPEN again (wait more)
```

```
Circuit Breaker States:

CLOSED ──(failures exceed threshold)──→ OPEN
  ↑                                        ↓
  └──(test request succeeds)── HALF-OPEN ←─┘
                                   ↓
                         (test request fails)
                                   ↓
                                 OPEN (reset timer)
```

**In Go with the popular `sony/gobreaker` library:**
```go
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 1,              // 1 test request in HALF-OPEN
    Interval:    60 * time.Second, // Reset counts every 60s
    Timeout:     30 * time.Second, // Stay OPEN for 30s before trying
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        // Open circuit if >50% of last 5 requests failed
        return counts.Requests >= 5 && 
               counts.TotalFailures/counts.Requests > 0.5
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return paymentClient.ProcessPayment(ctx, req)
})
```

---

## Planned vs Unplanned Downtime

**Not all downtime is created equal.**

| Type | Example | User Experience |
|------|---------|----------------|
| Unplanned | Server crash, bug, network failure | Sudden error, unexpected |
| Planned | Deployment, DB migration, maintenance | Can be scheduled at low-traffic time |

**SLAs usually count BOTH.** So a midnight deployment that takes the system down for 30 minutes still eats into your 99.9% budget (43.8 min/month).

### Zero-Downtime Deployment (Why It Matters)

```
Without zero-downtime:
  Deploy new version → stop old app → start new app → 2 min down
  
  If you deploy 3 times per week:
  3 × 2 min × 52 weeks = 312 minutes = 5.2 hours of planned downtime/year
  → Blows your 99.9% SLA (which only allows 8.76 hours/year total)

With zero-downtime (rolling deployment):
  Step 1: Add new version servers (say, 2 new + 3 old)
  Step 2: Load balancer sends traffic to all 5
  Step 3: Remove old servers one at a time
  Step 4: 0 minutes downtime
```

**Blue-Green Deployment:**
```
Environment BLUE:  Running old version (currently serving traffic)
Environment GREEN: Deploy new version here (tested, not yet live)

Switch LB → point to GREEN

If GREEN has a bug:
  Instantly switch LB → back to BLUE
  → Rollback in seconds, not minutes
```

**Your CI/CD context:** Your Docker + CI/CD pipeline at SAP probably does rolling or blue-green deployments. That's not just DevOps — it's what allows you to deploy frequently without burning your availability budget.

---

## SLA in a Team Lead Context

When you become Team Lead, you'll need to:

### 1. Define SLAs for your team's services
```
Questions to ask:
  - Who depends on this service? (other teams, customers)
  - What's the cost of 1 minute of downtime? (business impact)
  - What's the maximum acceptable p99 latency?
  
  Example SLA document for SAP BTP Terraform Provider API:
    Availability: 99.9%
    Latency: p99 < 500ms for state read, p99 < 2s for state write
    Error rate: < 0.1% of requests
    Data durability: 99.9999999% (S3 for state storage)
```

### 2. Track SLIs (Service Level Indicators) — the actual measurements

```
SLI: "What percentage of requests in the last 30 days returned 2xx?"
  Measure this with metrics (Prometheus, CloudWatch)
  If SLI drops below SLA threshold → alert fires

Your monitoring check:
  availability_sli = successful_requests / total_requests
  if availability_sli < 0.999:  # below 99.9%
    alert("SLA breach risk!")
```

### 3. Have an SLO (Service Level Objective) as an internal target

```
SLA: 99.9% (promise to customers, contractual)
SLO: 99.95% (your internal target, buffer above SLA)

The gap (0.05%) is your "error budget"
If you're at 99.92% (below SLO but above SLA):
  → Warning: slow down deployments, focus on stability
  
If you're at 99.88% (below SLA):
  → Emergency: stop new features, fix reliability NOW
```

---

## A Real Incident Story (To Internalize)

Let's say your ICICI Lombard insurance portal has a 99.9% SLA (8.76 hours/year budget).

**January:** DB migration takes 45 minutes → 45 min downtime  
**March:** A bug crashes the app for 2 hours → 2 hours downtime  
**June:** Deployment gone wrong → 1 hour downtime

Total so far: 3 hours 45 minutes used  
Remaining for rest of year: 4 hours 59 minutes

**July** comes. You want to deploy a big new feature. Test environment shows it's fine.
But your deployment takes 20 minutes of downtime (rolling deploy wasn't set up).

**Question:** Can you afford it? You have 4:59 remaining.  
**Answer:** Technically yes, but you're now at 5:39 left for Aug, Sep, Oct, Nov, Dec.  
That's less than 1 hour per month. Any incident could breach the SLA.

**As a Team Lead, you now push for:**
1. Zero-downtime deployments (stop consuming availability budget on deployments)
2. Better monitoring (catch issues before they become 2-hour outages)
3. Circuit breakers (cascading failures are the most expensive outages)

---

## Summary — 5 Things to Remember

```
1. Availability = percentage of time your system works.
   99.9% = 8.76 hours downtime/year. That's not as much as it sounds.

2. Chained services multiply availabilities.
   3 services at 99.9% = 0.999³ = 99.7%. Plan for this.

3. MTTR matters as much as MTBF.
   Halving recovery time can add a whole "nine" to your availability.

4. 4 patterns for high availability:
   Redundancy → Health checks → Graceful degradation → Circuit breaker

5. Error budget = gap between SLO and SLA.
   Manage it like a budget. Deployments and incidents consume it.
```

---

## Quick Quiz

1. You have 3 services: API (99.99%), Auth (99.9%), Database (99.95%). What's the total availability of a request that goes through all three?

2. Your app was down for 5 hours in the last 30 days. What's your monthly availability? Does it meet a 99.9% SLA?

3. Your payment service calls a Fraud Detection service. Fraud Detection starts returning errors 80% of the time due to a bug. Without a circuit breaker, what happens to your payment service? With a circuit breaker, what happens?

*(Answers: 1. 0.9999 × 0.999 × 0.9995 = 99.84%. 2. 5 hours down in 720 hours = 99.31% — does NOT meet 99.9% which requires < 43.8 min/month. 3. Without: payment service threads fill up waiting for fraud check → payment service crashes too. With: circuit opens, fraud check is skipped (or default "not fraud"), payment service stays healthy.)*

---

*Next: [Day 4 — Back-of-Envelope Estimation]
