# System-design-notes
Notes for system design in detail with examples and question answers

## Why This Plan Exists
you won't just write code — you'll design systems your team builds.  
You'll be in design reviews, architecture discussions, and interviews that probe:
- Can you design systems at scale?
- Can you make trade-off decisions confidently?
- Can you communicate architecture clearly to your team and stakeholders?

This plan is calibrated to **your real experience**: Go services, AWS, Terraform, PostgreSQL/MongoDB, Redis, IAM systems, healthcare/insurance backends.

## 2-Month Roadmap

```
Month 1: Foundations + Core Building Blocks
├── Week 1: Scalability Fundamentals & Estimation
├── Week 2: Databases — Design, Replication, Sharding
├── Week 3: Caching, CDN, Load Balancers
└── Week 4: Message Queues, Async Patterns, Event-Driven Design

Month 2: Real Systems + Interview Practice
├── Week 5: APIs, Rate Limiting, Auth/AuthZ Systems
├── Week 6: Design Case Studies (URL Shortener, Feed, Search)
├── Week 7: Design Case Studies (Payments, Notification, IAM)
└── Week 8: Mock Interviews + System Review + Gap Filling
```

---

## Weekly File Index

| Week | File | Focus |
|------|------|-------|
| 1 | [week1_scalability_estimation.md](week1_scalability_estimation.md) | Horizontal/vertical scale, CAP, Back-of-envelope |
| 2 | [week2_databases.md](week2_databases.md) | SQL vs NoSQL, Replication, Sharding, Indexes |
| 3 | [week3_caching_lb_cdn.md](week3_caching_lb_cdn.md) | Redis patterns, Load balancers, CDN |
| 4 | [week4_messaging_async.md](week4_messaging_async.md) | SQS/Kafka, Event-driven, Saga pattern |
| 5 | [week5_api_auth_ratelimit.md](week5_api_auth_ratelimit.md) | REST/gRPC, OAuth2/JWT, Rate limiting |
| 6 | [week6_case_studies_1.md](week6_case_studies_1.md) | URL Shortener, News Feed, Search |
| 7 | [week7_case_studies_2.md](week7_case_studies_2.md) | Payment System, Notification, IAM |
| 8 | [week8_mock_review.md](week8_mock_review.md) | Mock interviews, gaps, Team Lead framing |

---

## Daily Time Commitment

- **Weekdays:** 45–60 min
- **Weekend:** 1.5–2 hrs (design practice / case study)
- **Total per week:** ~6–7 hrs

---

## How to Practice Each Topic

1. **Read** — concepts in the weekly file  
2. **Draw** — sketch the architecture on paper/whiteboard  
3. **Explain** — say it out loud as if presenting to your team  
4. **Challenge** — answer "what breaks at 10x traffic?"  
5. **Map back** — "where did I see this in work?"

---

## Interview Framework (use every design session)

```
1. Clarify Requirements (3–5 min)
   - Functional: what the system does
   - Non-functional: scale, latency, availability

2. Estimate Scale (2–3 min)
   - Users, QPS, storage, bandwidth

3. High-Level Design (10 min)
   - Draw boxes: clients, LB, services, DB, cache

4. Deep Dive (15 min)
   - Pick 2–3 critical components to expand

5. Trade-offs & Bottlenecks (5 min)
   - What fails first? What would you do differently?
```

---

