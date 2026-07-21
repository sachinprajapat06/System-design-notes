# Day 1 — Horizontal vs Vertical Scaling
### Week 1 | System Design | Sachin's Study Notes

---

## What Are We Learning Today?

When your app gets popular and more people use it, your server gets tired.
Today we learn **two ways to make your server handle more load** — and why one is almost always better in the real world.

Think of it like this: you run a small chai stall. On a normal day, one person (you) handles everything. But during Diwali, 500 people show up. What do you do?

- **Option A:** Hire a super-human version of yourself who works 10x faster (Vertical Scaling)
- **Option B:** Hire 10 more people and split the work (Horizontal Scaling)

That's it. That's the whole concept. Let's go deep.

---

## Vertical Scaling (Scale Up)

**The simple idea:** Make your ONE machine bigger and more powerful.

```
Before:
  Server: 2 CPU cores, 4 GB RAM
  
After vertical scaling:
  Server: 32 CPU cores, 128 GB RAM
```

### Real Life Analogy — The Super Chef

Imagine you own a restaurant. You have 1 chef. During busy season, you upgrade:
- Give him a bigger, faster stove
- Add more prep tables
- Give him better knives

That's vertical scaling. Same person, better tools.

### In Your World (Go + AWS)

When your BluePi PostgreSQL database was getting slow, the first fix was probably:
- Upgrade from `db.t3.medium` → `db.t3.xlarge` (more CPU and RAM on the RDS instance)
- No code changes needed
- Database just got faster because the machine is more powerful

### When Vertical Scaling Is Good
```
✓ You need a quick fix RIGHT NOW (no time to redesign)
✓ Your app is stateful (hard to run on multiple machines)
✓ You're at small scale (10K users, not 10M)
✓ Simple setup — 1 server, nothing to coordinate
```

### The 3 Problems With Vertical Scaling

**Problem 1: There's a ceiling**
```
The biggest AWS EC2 instance: u-24tb1.metal → 448 vCPUs, 24 TB RAM
Cost: ~$218/hour = $1.9 MILLION per year

But Google processes 8.5 billion searches/day.
No single machine in the world can handle that.
```

**Problem 2: Single Point of Failure (SPOF)**
```
Your 1 server goes down → your ENTIRE app is down.

If you upgraded your 1 chef and he gets sick → restaurant closes.
If you have 10 chefs and one gets sick → 9 others keep cooking.
```

**Problem 3: Downtime to Upgrade**
```
To upgrade from 16 GB RAM to 64 GB RAM on AWS RDS:
  - AWS stops the instance
  - Resizes it
  - Restarts it
  → Your app is down for 5–20 minutes

At BluePi with ICICI Lombard — a 20-minute downtime during business hours
means insurance agents can't process claims. That's a business impact.
```

---

## Horizontal Scaling (Scale Out)

**The simple idea:** Add more machines. Spread the work across all of them.

```
Before:
  1 server handling 1,000 requests/second

After horizontal scaling:
  5 servers, each handling 200 requests/second
  Together: still 1,000 requests/second
  
  Need more? Add 5 more servers → 10 servers × 200 = 2,000 req/sec
```

### Real Life Analogy — The Call Center

Think of a customer service call center. 
- They don't hire 1 superhuman agent who answers 1,000 calls simultaneously
- They hire 1,000 normal agents who each answer 1 call at a time
- If demand doubles → hire 1,000 more agents

That's horizontal scaling.

### In Your World — AWS Lambda

```
Your Lambda function at AWS is ALREADY horizontally scaled, automatically.
  
When 1 request comes in:
  AWS runs 1 Lambda instance to handle it
  
When 1,000 simultaneous requests come in:
  AWS spins up 1,000 Lambda instances in parallel
  Each handles 1 request
  
You never configured this. Lambda does it by default.
That's the power of horizontal scaling built into the platform.
```

### The Key Requirement: Stateless Services

This is the most important concept for horizontal scaling.

**Stateful** means the server remembers something about the user between requests.
**Stateless** means the server remembers NOTHING — each request is brand new.

```
STATEFUL (BAD for horizontal scaling):

  User logs in → Server A stores session in memory
  
  User's next request goes to Server B (round robin)
  Server B: "I don't know this user. Who are you?"
  → User gets logged out. Bug.

STATELESS (GOOD for horizontal scaling):

  User logs in → Session stored in REDIS (shared between all servers)
  
  User's next request goes to Server B
  Server B: checks Redis → "Ah, this is Sachin, logged in, here's his data"
  → Works perfectly, user never notices
```

### Where State Lives in a Horizontal System

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Server A   │    │  Server B   │    │  Server C   │
│  (stateless)│    │  (stateless)│    │  (stateless)│
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
              ┌───────────┴───────────┐
              │      Shared State      │
              │  Redis (sessions)     │
              │  PostgreSQL (data)    │
              │  S3 (files)          │
              └───────────────────────┘
```

All 3 servers are identical. Any server can handle any request. 
State lives outside the servers — in Redis, PostgreSQL, or S3.

**In your Go code, this means:**
- Don't store user data in a global variable
- Don't store anything in memory that should survive a server restart
- Always read from DB/Redis, always write back to DB/Redis

---

## Side by Side Comparison

| Feature | Vertical (Scale Up) | Horizontal (Scale Out) |
|---------|--------------------|-----------------------|
| How | Bigger machine | More machines |
| Complexity | Low (no code changes) | Higher (need LB, shared state) |
| Cost at small scale | Cheaper | More expensive |
| Cost at large scale | Very expensive (hardware ceiling) | Cheaper (commodity servers) |
| Max scale | Limited by hardware | Nearly unlimited |
| Failure risk | High (1 machine = 1 point of failure) | Low (1 machine fails, others work) |
| Downtime to scale | Yes (restart needed) | No (add servers with zero downtime) |
| Your AWS Lambda | ✗ | ✓ (auto horizontal) |
| Your PostgreSQL (BluePi) | ✓ (started here) | Later (read replicas, then sharding) |

---

## Real World Example: Instagram's Growth

When Instagram launched (2010):
- 1 server
- Vertical scaling for first few months
- Hit the wall at ~100K users

Then they switched to horizontal:
- Multiple app servers behind a load balancer
- PostgreSQL with read replicas
- Memcached (cache layer) shared across all servers

**Key lesson:** Almost every big system starts vertical (simple, cheap, fast to launch) and eventually moves horizontal (scales beyond what 1 machine can do).

---

## What This Means in a Team Lead Role

When you're a Team Lead and a junior engineer says:
> "Our API is getting slow, should we upgrade the server?"

You now ask the right questions:
```
1. "How many users do we have? What's the growth rate?"
   → If it's growing fast, vertical will only delay the problem

2. "Is our service stateless?"
   → If not, horizontal scaling needs work first

3. "What's the actual bottleneck — CPU, memory, or database?"
   → Vertical helps CPU/memory; DB is often the real bottleneck

4. "What's the cost of downtime for a resize?"
   → If SLA is 99.9%, planned downtime must be within that budget
```

This is the difference between a junior who says "make it bigger" and a lead who says "let's understand what's actually bottlenecking us."

---

## Summary — 5 Things to Remember

```
1. Vertical = bigger machine. Horizontal = more machines.

2. Horizontal scales to any size. Vertical hits a hardware ceiling.

3. Horizontal requires STATELESS services — state lives in Redis/DB.

4. Horizontal gives you fault tolerance — 1 machine dies, others serve traffic.

5. In the real world: start vertical (cheap, fast), switch to horizontal 
   when you're growing and need reliability.
```

---

## Quick Quiz (answer in your head)

1. Your Go service stores user session data in a Go `map` in memory.  
   Can you horizontally scale it? Why not? What do you fix?

2. Your Lambda function is suddenly getting 50,000 concurrent requests.  
   Does AWS need you to "add more servers"? Why?

3. Your RDS PostgreSQL is overwhelmed by 10,000 reads/second.  
   What do you try first — vertical or horizontal? What's the horizontal solution for a DB?

*(Answers: 1. No — move session to Redis. 2. No — Lambda auto-scales horizontally. 3. Vertical first for quick fix; horizontal = read replicas.)*

---

*Next: [Day 2 — CAP Theorem](day2_cap_theorem.md)*
