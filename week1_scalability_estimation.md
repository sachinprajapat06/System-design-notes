# Week 1: Scalability Fundamentals & Estimation

**Goal:** Build mental models for *how systems grow* and *how to estimate their needs.*

---

## Day 1 — Horizontal vs Vertical Scaling

### Vertical Scaling (Scale Up)
- Upgrade the machine: more CPU, more RAM, bigger disk
- Simple — no code changes needed
- **Limit:** hardware ceiling + single point of failure
- **Your example:** Your PostgreSQL on BluePi likely ran on a single RDS instance — that's vertical

### Horizontal Scaling (Scale Out)
- Add more machines, distribute the load
- Requires stateless services (no session on server)
- **Your example:** Lambda functions at AWS scale horizontally automatically — each invocation is isolated
- Trade-off: now you need a Load Balancer, shared state (Redis/DB), and distributed coordination

### Rule of Thumb
```
Vertical:  Fast win, cheap at small scale, hits a wall
Horizontal: Harder to set up, scales indefinitely, is where real production systems live
```

---

## Day 2 — CAP Theorem

Every distributed system can guarantee only **2 of 3**:

| Property | Meaning |
|----------|---------|
| **C**onsistency | Every read sees the latest write |
| **A**vailability | Every request gets a response (not necessarily latest data) |
| **P**artition Tolerance | System works even if nodes can't talk to each other |

**Network partitions always happen** → you must choose C or A when a partition occurs.

### Real Choices You've Made (Without Knowing It)

| System | Choice | Why |
|--------|--------|-----|
| PostgreSQL (BluePi) | CP | ACID transactions — correctness over availability |
| DynamoDB (AWS) | AP | Eventually consistent by default, tunable |
| Redis Cluster | AP (default) | High availability, async replication |
| Cisco IAM token store | CP | Wrong token = security breach |

### PACELC Extension (more realistic)
When no partition:
- **Latency vs Consistency** trade-off still exists
- DynamoDB: low latency even if slightly stale data  
- PostgreSQL with synchronous replication: consistent but higher write latency

---

## Day 3 — Availability & SLAs

### Nines of Availability

| SLA | Downtime/Year | Used For |
|-----|--------------|----------|
| 99% | 3.65 days | Internal tools |
| 99.9% | 8.7 hours | Most web services |
| 99.99% | 52 minutes | Payment systems |
| 99.999% | 5 minutes | Telecom, banking core |

### How to Achieve High Availability
1. **Redundancy** — no single point of failure
2. **Health checks + auto-restart** — your Docker/K8s already does this
3. **Graceful degradation** — serve stale cache if DB is down
4. **Circuit breaker** — stop hammering a failing service

**Your Cisco IAM parallel:** An IAM system at 99.9% means 8.7 hrs downtime/year — unacceptable for auth. At Cisco, what SLA did your IAM service target? That's a real design constraint.

---

## Day 4 — Back-of-Envelope Estimation

The goal is not precision. The goal is **order of magnitude** — are we talking kilobytes or terabytes?

### Key Numbers to Memorize

```
1 million requests/day = ~12 requests/second
1 billion requests/day = ~12,000 requests/second (12K QPS)

1 KB  = 1,000 bytes
1 MB  = 1,000,000 bytes
1 GB  = 10^9 bytes
1 TB  = 10^12 bytes

Read from memory:    ~100 ns
Read from SSD:       ~100 µs  (1,000x slower)
Read from HDD:       ~10 ms   (100,000x slower)
Network roundtrip:   ~150 ms  (US cross-region)
```

### Estimation Template

```
1. Daily Active Users (DAU): e.g., 10 million
2. Requests per user/day: e.g., 10 reads, 2 writes
3. Total QPS = (DAU × requests) / 86,400 seconds
4. Peak QPS = avg QPS × 2-3x
5. Storage = new data per day × retention period
6. Bandwidth = avg request size × QPS
```

### Practice: Estimate Your SAP BTP Terraform Provider

- How many Terraform apply operations per day across all SAP customers?
- How large is a typical state file?
- How many API calls does each `terraform apply` make to SAP BTP?
- What's the peak window (business hours, deployments)?

This is the kind of reasoning that impresses in a lead role — you connect scale to real systems.

---

## Day 5 — DNS, Load Balancers, and CDN Overview

*(Deep dive in Week 3 — just build the mental model today)*

```
User → DNS → Load Balancer → App Servers → Database
                  ↓
              CDN (for static assets)
```

### DNS
- Translates domain → IP
- TTL matters: low TTL = fast failover but more DNS queries
- Your Terraform Provider might set DNS records for SAP BTP services

### Load Balancer Types
| Type | Works at | Example |
|------|---------|---------|
| L4 (Network) | TCP/UDP | AWS NLB |
| L7 (Application) | HTTP/HTTPS | AWS ALB, Nginx |

### Algorithms
- **Round Robin** — simple, even distribution
- **Least Connections** — route to server with fewest active connections
- **IP Hash** — same client always hits same server (sticky sessions)

---

## Weekend Practice — Week 1

### Exercise 1: Draw "Design a URL Shortener at 100M users"
Just the boxes — don't optimize yet. Answer:
- Where does it scale horizontally?
- What's the bottleneck?

### Exercise 2: Estimate
A healthcare app like your BluePi work: 500K patients, each uploads one document/month (avg 2 MB).
- Monthly storage added?
- Annual storage?
- S3 cost at $0.023/GB/month?

### Exercise 3: CAP Decision Log
For each system you've worked on, write: System → CAP choice → Why it was right

---

## Key Takeaways — Week 1

- Horizontal scaling is the default for production — stateless services + shared state
- CAP theorem: in a partition, you choose C or A. Know which your system chose and why
- Back-of-envelope is a leadership skill — you need to challenge estimates in design reviews
- Availability SLAs drive architecture decisions, not just code quality

---
