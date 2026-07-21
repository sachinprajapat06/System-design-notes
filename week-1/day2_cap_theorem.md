# Day 2 — CAP Theorem
### Week 1 | System Design | Sachin's Study Notes

---

## What Are We Learning Today?

When you have multiple computers working together (a distributed system), something very fundamental happens: **you can never have everything you want at the same time.**

The CAP Theorem is the rule that explains this limitation. It was proven mathematically by Eric Brewer in 2000 and it applies to EVERY distributed system — including every system you've built.

The wild part? **You've already been making CAP decisions** every time you chose PostgreSQL over DynamoDB or Redis over a database. You just didn't know it had a name.

---

## The Simple Explanation First

Imagine you have a **notebook** where you write down prices. You have TWO shops (Shop A and Shop B) and both shops have a copy of this notebook.

Now, three things could happen:

1. **Consistency (C):** Both shops always show the same price. If you update the price in Shop A, Shop B immediately knows about it too. You always get "the truth."

2. **Availability (A):** Both shops are always open and always give you SOME answer. Even if the phone line between them is cut, each shop stays open and serves customers.

3. **Partition Tolerance (P):** The shops can still work even if the phone line between them breaks (network partition = network failure between nodes).

**The Theorem says:** When the phone line breaks (and it WILL break — networks are unreliable), you can only pick:
- Stay consistent but possibly go offline for some shop (C + P)
- Stay open but possibly give different/stale prices (A + P)

**You cannot have C + A + P simultaneously.** That's the CAP Theorem.

---

## The Three Properties in Detail

### C — Consistency

**What it means:** Every read returns the most recent write, OR returns an error.

There's no in-between. If you wrote value "100" to node A, and then immediately read from node B, you will EITHER get "100" or get an error. You will NEVER get stale data.

```
Timeline:
  T1: Client writes balance = 500 to Node A
  T2: Node A replicates to Node B (takes 50ms)
  T3: Client reads balance from Node B

  With CONSISTENCY:
    If T3 happens at T2 + 10ms (before replication): Node B returns ERROR
    or Node B waits until replication completes, then returns 500
    → You either get correct data or an error. Never stale.
    
  Without CONSISTENCY (eventual consistency):
    Node B returns 400 (the old value)
    → You got stale data. For a bank balance, that's dangerous.
```

**When consistency is critical:**
- Bank balances (if two ATMs both show ₹10,000, both let you withdraw → you overdraw)
- Inventory systems (if two systems both show "1 item left", both sell it → oversell)
- Authentication tokens (if token is revoked but old copy is served → security breach)

**Your real example — Cisco IAM:**
Your IAM token store is CP (Consistency + Partition tolerance). If a token is revoked (user logged out or account compromised), **every node must immediately stop accepting that token**. If some node returns a cached "valid" when it should say "revoked", that's a security vulnerability.

### A — Availability

**What it means:** Every request receives a response (not necessarily the latest data), without error.

The system is always up. It will always respond. But the response might be slightly outdated.

```
Timeline:
  T1: Client writes balance = 500 to Node A
  T2: Replication to Node B is delayed (network slow)
  T3: Client reads balance from Node B

  With AVAILABILITY:
    Node B responds immediately: 400 (old value)
    → Client got a response (no error), but it's stale
    → System is always available, data is eventually consistent
```

**When availability is more important:**
- Social media "likes" count (showing 9,998 instead of 10,000 for 50ms is fine)
- Product catalog (seeing yesterday's price briefly is acceptable)
- Your shopping cart (losing 100ms of sync is fine, cart being down is not)

**Your real example — DynamoDB on AWS:**
DynamoDB by default is AP. When you do a `GetItem`, you might get slightly stale data if a write just happened. But DynamoDB will ALWAYS respond, never go down. For Lambda + DynamoDB use cases (e.g., job queues, session data), brief staleness is fine.

### P — Partition Tolerance

**What it means:** The system continues to work even when network messages between nodes are lost or delayed.

```
Node A ←—— ✗ network cut ——→ Node B

A partition = nodes can't talk to each other.
This WILL happen in production. Networks fail. Cables are cut. 
AWS availability zones lose connectivity. Cloud providers have incidents.
```

**Here's the truth about P:** In a distributed system, **you MUST tolerate partitions**. You can't "turn off" partition tolerance — because partitions happen whether you want them to or not.

So CAP really becomes: **when a partition happens, do you sacrifice C or A?**

---

## The Real Choice: CP vs AP

```
┌─────────────────────────────────────────────┐
│  Network partition occurs between Node A    │
│  and Node B                                 │
│                                             │
│  You must choose:                           │
│                                             │
│  Option 1 (CP):                             │
│    Refuse to serve requests until nodes     │
│    can sync. System may appear "down"       │
│    but data is ALWAYS correct.              │
│                                             │
│  Option 2 (AP):                             │
│    Keep serving requests from each node.    │
│    System is always "up" but nodes may      │
│    return different (stale) data.           │
└─────────────────────────────────────────────┘
```

---

## Real Systems and Their CAP Choices

### PostgreSQL — CP

```
Why CP?
  PostgreSQL with synchronous replication:
  - Primary writes and waits for replica to confirm
  - If replica is unreachable (partition): 
    Primary can refuse writes (to avoid diverging state)
  - Data is always consistent
  
  This is why you use PostgreSQL for:
  - Insurance policies (ICICI Lombard)
  - Financial records
  - User accounts with passwords
  
  A billing record showing wrong amount = legal problem
  Better to be temporarily unavailable than permanently wrong
```

### DynamoDB — AP (by default, tunable)

```
Why AP?
  DynamoDB by default uses eventual consistency:
  - Read might return value from 100ms ago
  - But DynamoDB is ALWAYS available (99.99% SLA)
  - Never returns an error just because of replication lag
  
  DynamoDB CAN do strong consistency:
  - ConsistentRead: true in your GetItem call
  - Reads from the leader node only
  - Slower, costs 2x read capacity units
  - But now you have CP behavior
  
  Your AWS Lambda + DynamoDB work:
  - Default: AP (fast, cheap, eventually consistent)
  - For critical data: ConsistentRead: true (CP)
```

### Redis Cluster — AP (default), CP (with WAIT command)

```
Redis Cluster uses async replication by default:
  Primary writes → immediately returns OK to client
  → THEN asynchronously replicates to replicas
  
  If primary crashes before replication:
    New primary (promoted replica) might be missing last few writes
    → Data loss = NOT consistent
    → But Redis was always available = AP
  
  For stricter consistency: use WAIT numreplicas timeout
    Forces Redis to wait until N replicas confirm before returning OK
    → Now it's CP behavior
    
  At Cisco IAM for session tokens: you want CP for auth data
  For rate limiting counters: AP is fine (being off by 1 is acceptable)
```

### Cassandra — AP

```
Cassandra is designed for massive scale and availability:
  - Multiple nodes, any node can handle any request
  - Writes are accepted by any node, propagated later
  - Tunable consistency: you choose per-query (ONE, QUORUM, ALL)
  
  ONE:    Fastest, least consistent (accept from 1 node)
  QUORUM: Most common (majority of nodes must agree)
  ALL:    Slowest, most consistent (all nodes must agree)
```

---

## The Confusion About "CA" Systems

You might see databases listed as "CA" (Consistent + Available, no Partition Tolerance).

```
Wait — doesn't the theorem say you can only pick 2?

CA systems are:
  - Single-node databases (no distribution = no partition problem)
  - Systems that CHOOSE to stop working when a partition happens

A standalone PostgreSQL with NO replicas is "CA":
  - Consistent (single node, no replication lag)
  - Available (always responding)
  - Not partition tolerant (but there's nothing to partition — it's 1 machine)

The moment you add a read replica to PostgreSQL:
  - Now you have 2 nodes
  - Now partitions can happen
  - Now you must choose C or A
  - You're in CP territory
```

---

## PACELC — The Realistic Extension

CAP only talks about what happens **during a partition**. But what about **normal operation** (no partition)?

PACELC adds the next question:
- **PAC:** During a Partition: choose Availability or Consistency
- **ELC:** Else (no partition): choose Latency or Consistency

```
┌──────────────┬──────────────┬────────────────┐
│ System       │ During Partition │ Normal Operation│
├──────────────┼──────────────┼────────────────┤
│ DynamoDB     │ AP           │ Low Latency    │
│ PostgreSQL   │ CP           │ Consistency    │
│ Cassandra    │ AP           │ Low Latency    │
│ MongoDB      │ CP           │ Consistency    │
│ MySQL        │ CP           │ Consistency    │
└──────────────┴──────────────┴────────────────┘
```

**Real example:**
DynamoDB: Even without a partition, it returns fast (EL = low latency chosen over strong consistency). PostgreSQL: Even without a partition, it waits for synchronous write confirmation (EC = consistency chosen over latency).

---

## A Story That Makes This Concrete

**The WhatsApp Message Problem:**

You send "I love you ❤️" to your partner. WhatsApp uses an AP system.

```
Scenario 1 (AP):
  Your message goes to Server A
  Network partition happens
  Server A and Server B can't sync
  
  Your partner reads from Server B → doesn't see your message yet
  But Server B is still UP and responding ✓
  
  After 2 seconds, partition heals, message arrives
  "I love you ❤️" shows up
  → Briefly inconsistent, but both servers were always available
  → Acceptable! Being 2 seconds late is fine.

Scenario 2 (CP — what if WhatsApp was consistent?):
  Network partition happens
  Server B refuses to serve messages until it syncs with Server A
  Your partner opens WhatsApp → gets "Server unavailable" error
  
  → Consistent! But your partner thinks WhatsApp is broken.
  → For chat messages, consistency matters less than availability.
```

Now flip it to banking:

```
You send ₹50,000 to your friend for a house deposit.

Scenario 1 (AP — BAD for banking):
  Bank Server A processes the debit (you -₹50,000)
  Network partition
  Bank Server B doesn't know yet
  Friend reads balance from Server B → shows ₹0 received
  Friend can't pay the deposit → loses the house
  → Availability was there but consistency wasn't → real harm

Scenario 2 (CP — CORRECT for banking):
  Bank Server A processes debit
  Network partition
  Bank Server B refuses reads → "System temporarily unavailable"
  Friend waits 30 seconds → partition heals → ₹50,000 shows up
  → Brief unavailability → no harm done
```

**Key insight:** The right choice depends on what your data means and what "wrong" costs.

---

## How to Decide CP vs AP for YOUR System

Ask this question: **"What's worse — being briefly wrong, or being briefly unavailable?"**

```
User account password store → CP
  Wrong: someone logs in with old password = security breach
  Unavailable: login fails for 5 seconds = annoying but safe
  → CP is correct

Instagram like count → AP
  Wrong: shows 9,999 instead of 10,000 for 50ms = nobody cares
  Unavailable: Instagram shows error = users complain, engagement drops
  → AP is correct

Hospital patient medication record → CP
  Wrong: doctor sees old medication → prescribes wrong drug = death
  Unavailable: doctor sees error, double-checks manually = safe delay
  → CP, strongly

Shopping product inventory → depends on context
  For "1 item left" → CP (overselling is a real problem)
  For "1,000 items in stock" → AP (being off by 1 doesn't matter)
```

---

## Mapping to Your Past Systems

```
System                  | Your Choice | Was it Right?
------------------------|-------------|----------------------------
PostgreSQL (ICICI)      | CP          | Yes — insurance records must be correct
MongoDB (RXBenefits)    | CP default  | Yes — healthcare data needs accuracy
DynamoDB (AWS Lambda)   | AP default  | Yes — job queues, slight delay OK
Redis (Cisco IAM)       | AP → CP     | Depended on data:
                        |             |   Sessions: CP (security)
                        |             |   Rate limit counters: AP (OK to be off)
SAP BTP Terraform state | CP          | Yes — wrong state = broken infra
```

---

## Summary — 5 Things to Remember

```
1. CAP = Consistency, Availability, Partition Tolerance.
   You can only pick 2. In practice: you always have P, so choose C or A.

2. CP = When a partition happens, the system might refuse requests 
   but never returns wrong data.
   Use for: financial data, auth tokens, healthcare records.

3. AP = When a partition happens, the system keeps responding
   but might return stale data.
   Use for: social feeds, product catalogs, analytics counters.

4. This is a SPECTRUM, not a binary choice.
   DynamoDB lets you tune per-query (ConsistentRead: true/false).
   Cassandra lets you tune per-query (QUORUM vs ONE).

5. You've been making CAP decisions your whole career.
   PostgreSQL for ICICI = CP. DynamoDB for Lambda = AP.
   Now you can explain WHY.
```

---

## Quick Quiz

1. Your insurance claim system uses MongoDB. A replica falls behind. Your app reads from the replica and shows the claim as "Pending" when it's actually "Approved." Is MongoDB behaving as CP or AP here? Should you change this?

2. You're designing a rate limiter for the SAP BTP API. Each server checks a Redis counter. Redis is AP. Two servers both allow a request because they both see "count: 99" before one updates it to 100. Is this a problem? What's the consequence? Is AP acceptable here?

3. You have a 2-node Redis cluster. Node 1 accepts a "user_123 logged out" write. Before it replicates to Node 2, Node 1 crashes. Node 2 becomes primary. A request comes in using user_123's token — Node 2 still shows them as logged in. CP or AP behavior? Is this a problem?

*(Answers: 1. AP behavior — use readPreference: primary for critical reads. 2. Rate limiter slightly over-counts, allows 101 instead of 100 — minor, AP is usually fine here. 3. AP — yes, this is a security problem for auth tokens; use synchronous replication or short JWT TTL as mitigation.)*

---

*Next: [Day 3 — Availability & SLAs]
