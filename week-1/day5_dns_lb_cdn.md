# Day 5 — DNS, Load Balancers & CDN Overview
### Week 1 | System Design | Sachin's Study Notes

---

## What Are We Learning Today?

Before a request reaches your Go service, it travels through **at least 3 invisible systems** that most developers never think about:

1. **DNS** — finds the IP address of who to talk to
2. **CDN** — serves static files from the closest server in the world
3. **Load Balancer** — decides which of your servers handles the request

These aren't just infrastructure concerns — they're architectural decisions that affect latency, availability, cost, and scalability. Today is an introduction (we go deep in Week 3). The goal today is to build a mental map of the full journey a request takes.

---

## The Full Request Journey

Let's trace what happens when a user in Mumbai types `sap.com` in their browser:

```
Step 1: Browser checks its own DNS cache
  "Do I remember the IP for sap.com?"
  → No, last time was yesterday and TTL expired

Step 2: Ask local DNS resolver (ISP or Google's 8.8.8.8)
  "What's the IP for sap.com?"
  → 192.0.2.1 (SAP's load balancer IP)

Step 3: Request goes to CDN edge node
  "Do I have /static/logo.png cached?"
  → Yes! Return it from CDN in Mumbai → 5ms ✓
  
  "Do I have the API response for this query?"
  → No → forward to origin server

Step 4: Request reaches Load Balancer
  "Which of my 10 app servers should handle this?"
  → Picks Server #3 (least loaded)

Step 5: Server #3 processes the request
  → Reads from Redis, queries PostgreSQL
  → Returns response

Step 6: Response travels back
  Server #3 → Load Balancer → (CDN caches it if appropriate) → User

Total for API: ~80-150ms
Total for static asset (CDN hit): ~5-20ms
```

Now let's understand each component deeply.

---

## Part 1: DNS (Domain Name System)

### The Simplest Explanation

DNS is the **phone book of the internet.**

You know your friend's name ("Rahul") but not their phone number. You look them up in the phone book to find the number. Similarly, you know the domain name ("sap.com") but not the IP address. DNS gives you the IP.

```
sap.com  →  DNS lookup  →  192.0.2.1

Without DNS: you'd have to type 192.0.2.1 in your browser (impossible to remember)
With DNS: you type sap.com, DNS resolves it to an IP address automatically
```

### How DNS Resolution Works (The Full Chain)

```
Your browser → checks local cache (built into browser)
  ↓ (cache miss)
Your OS → checks /etc/hosts file  
  ↓ (not found)
Your OS → asks your DNS resolver (usually your router or ISP)
  ↓ (cache miss)
DNS Resolver → asks Root DNS servers (13 servers worldwide)
  Root says: "I don't know sap.com but for .com domains, 
               ask Verisign's TLD servers at these IPs"
  ↓
DNS Resolver → asks .com TLD server (Verisign)
  TLD says: "For sap.com, ask SAP's authoritative DNS server at X.X.X.X"
  ↓
DNS Resolver → asks SAP's authoritative DNS server
  SAP's DNS says: "sap.com is at 192.0.2.1"
  ↓
DNS Resolver → caches the answer → returns 192.0.2.1 to your browser
  ↓
Your browser → makes HTTP request to 192.0.2.1
```

This looks slow but in reality it takes 20-100ms and is cached aggressively.

### TTL (Time To Live) — The Most Important DNS Concept

Every DNS record has a TTL — how long DNS resolvers and browsers should cache the answer.

```
sap.com IN A 192.0.2.1  TTL=3600

This means: cache this mapping for 3600 seconds (1 hour)
After 1 hour: browsers and resolvers re-check for fresh record

Low TTL (60 seconds):
  Pro: Changes propagate fast (failover in 60 seconds)
  Con: More DNS queries = higher DNS server load, slight overhead

High TTL (3600 seconds = 1 hour):
  Pro: Fewer DNS queries, faster (cache hit)
  Con: If you change IPs, old IP is used for up to 1 hour globally
```

**Real world scenario:**

You're doing maintenance on your SAP BTP service. You want to fail over to a backup server:

```
Strategy:
  Normal operation: TTL = 3600 seconds (1 hour)
  
  BEFORE maintenance (30 minutes before):
    Lower TTL to 60 seconds
    Wait 1 hour (let all caches expire with old high TTL)
    
  During maintenance:
    Change DNS record to point to backup server
    Within 60 seconds: all users hitting new server ✓
    
  After maintenance:
    Change DNS record back to primary server
    Raise TTL back to 3600 seconds
```

If you didn't lower TTL first: some users would be pointing at the old/wrong IP for up to 1 hour.

### DNS Record Types You Need to Know

```
A Record:     domain → IPv4 address
  sap.com    →  192.0.2.1

AAAA Record:  domain → IPv6 address
  sap.com    →  2001:db8::1

CNAME:        domain → another domain (alias)
  www.sap.com →  sap.com (www is just an alias for the root)
  
  Useful for: pointing multiple subdomains to same destination
              changing the destination by updating only one place

MX Record:    which server handles email for this domain
  sap.com    →  mail.sap.com (SAP's email server)

TXT Record:   arbitrary text, used for verification/security
  sap.com    →  "v=spf1 ..."  (SPF record — prevents email spoofing)

NS Record:    which DNS servers are authoritative for this domain
  sap.com    →  ns1.sap.com, ns2.sap.com

Your Terraform context:
  The SAP BTP Terraform Provider likely manages DNS records.
  When you provision a new environment, Terraform creates 
  an A record pointing the new subdomain to the environment's IP.
```

### GeoDNS — DNS-Level Geographic Routing

Advanced DNS can return DIFFERENT IPs based on where the user is:

```
User in India   → queries sap.com → gets IP: 103.20.X.X (Mumbai data center)
User in Germany → queries sap.com → gets IP: 193.30.X.X (Frankfurt data center)
User in USA     → queries sap.com → gets IP: 52.10.X.X  (AWS us-east-1)

Benefit: Users always routed to nearest data center → lower latency
This is how global companies serve regionally with 1 domain name
```

---

## Part 2: Content Delivery Network (CDN)

### The Problem CDN Solves

Your server is in AWS `ap-south-1` (Mumbai). A user in Germany requests your CSS file.

```
Without CDN:
  User in Germany → HTTP request → Mumbai (15,000 km away) → response back
  Latency: ~150ms for each static file
  
  Your home page loads: HTML + CSS + JS + 20 images
  If each needs a round-trip to Mumbai: 23 files × 150ms = 3.45 seconds
  → Very slow for European users

With CDN:
  First request (from any European user):
    CDN edge in Frankfurt → cache miss → fetches from Mumbai → caches it
    Latency: still 150ms for this first user
    
  Every subsequent request from Europe:
    CDN edge in Frankfurt → cache hit → returns instantly
    Latency: ~5ms (Frankfurt is 50km from many German cities)
    
  → 30x faster for all European users after the first request
```

### How CDN Works

```
                        ┌─────────────────────┐
                        │   Your Origin Server │
                        │   (AWS Mumbai)       │
                        └──────────┬──────────┘
                                   │ (cache miss only)
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
      ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
      │  CDN Edge     │   │  CDN Edge     │   │  CDN Edge     │
      │  (Mumbai)     │   │  (Frankfurt)  │   │  (New York)   │
      └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
              │                   │                   │
              ▼                   ▼                   ▼
     Indian users          European users      American users
     ~5ms latency          ~5ms latency        ~5ms latency
```

CDN providers (CloudFront, Cloudflare, Akamai) have **hundreds of edge nodes** globally. Content is cached at the nearest node to users.

### What to Put on CDN

**Good candidates (cache aggressively):**
```
✓ Static files: CSS, JavaScript, fonts, favicon
✓ Images: product photos, user avatars, thumbnails
✓ Videos: with proper streaming (HLS segments)
✓ Documents: PDFs, downloadable files
✓ API responses that are the same for all users (product catalog, configuration)

Example Cache-Control header for static JS file:
  Cache-Control: public, max-age=31536000, immutable
  (cache for 1 year, never changes because filename includes content hash)
```

**Bad candidates (don't cache):**
```
✗ User-specific data: account info, personalized feeds
✗ Frequently changing data: live stock prices, real-time scores
✗ Authenticated responses: things requiring auth headers
✗ Write requests: POST/PUT/DELETE (CDN is for reads only)

Example Cache-Control for user-specific API:
  Cache-Control: private, no-cache
  (browser can cache but CDN must not)
```

### CDN Cache-Busting

```
Problem: You deploy new CSS. Users still get old CSS from CDN cache.

Bad solution: short TTL (1 hour)
  → CDN re-fetches every hour even when nothing changed
  → Higher origin load, slower for users (more cache misses)

Good solution: Hash in filename
  Old file: styles.css     → cached with 1-year TTL
  New file: styles.a3b9c1.css → brand new URL → CDN has no cache for it
                                → fetches fresh from origin automatically
  
  The hash (a3b9c1) is the hash of the file content.
  Content changes → hash changes → filename changes → no cache to bust.
  
  Your build tool (Webpack, Vite, etc.) does this automatically.
```

### CDN for SAP Context

```
SAP BTP has SAP UI5 framework files (JavaScript).
These are served via CDN: https://ui5.sap.com/resources/...

When a BTP application loads:
  → Browser fetches UI5 from CDN (fast, globally cached)
  → Browser fetches app-specific code from SAP server
  → Browser fetches your custom SAP app from SAP customer's server

This 3-tier CDN pattern means:
  Heavy UI framework files: from CDN, very fast globally
  Business data: from your specific server (can't be CDN cached)
```

---

## Part 3: Load Balancers

### The Problem Load Balancers Solve

You have 5 servers. A request comes in. Which server should handle it?

```
Without a load balancer:
  Clients know the IP of all 5 servers?
  Clients pick randomly? What if Server 3 is overloaded?
  What if Server 4 crashes — clients keep sending to it?
  
  → Chaos. Clients can't manage server selection.

With a load balancer:
  Clients know ONE IP: the load balancer's
  Load balancer knows all 5 servers
  Load balancer distributes requests intelligently
  If Server 4 crashes: load balancer detects via health check, stops routing there
  → Clients are unaffected by individual server failures
```

### Layer 4 vs Layer 7 Load Balancers

This is the most important distinction.

**Layer 4 (Transport Layer — TCP/IP)**
```
Works at: TCP level (doesn't see HTTP content)
Sees: Source IP, Destination IP, Port
Doesn't see: URL path, HTTP headers, cookies

Example: AWS Network Load Balancer (NLB)

How it routes:
  Client connects to LB via TCP
  LB forwards TCP connection to a backend server
  Backend directly handles the HTTP/HTTPS
  
Use when:
  - Ultra-low latency needed (L4 has less overhead than L7)
  - Non-HTTP traffic (raw TCP, UDP, gaming servers)
  - Very high connection counts (millions per second)
```

**Layer 7 (Application Layer — HTTP/HTTPS)**
```
Works at: HTTP level (reads and understands HTTP requests)
Sees: URL path, headers, cookies, body
Can: route based on URL, add headers, terminate SSL, rewrite URLs

Example: AWS Application Load Balancer (ALB), Nginx

How it routes:
  GET /api/users → route to User Service
  GET /api/products → route to Product Service
  Static files → route to S3/CDN
  
Use when:
  - Microservices (route by URL path)
  - SSL termination (decrypt HTTPS at LB, send HTTP to servers)
  - Path-based routing
  - Content-based routing
  - A/B testing (route 10% to new version)
```

```
┌─────────────────────────────────────────────────────────┐
│ Feature            │    L4 (NLB)    │    L7 (ALB)       │
├─────────────────────────────────────────────────────────┤
│ Routing basis      │ IP + Port      │ URL, headers      │
│ SSL termination    │ Pass-through   │ Yes (at LB)       │
│ Path-based routing │ No             │ Yes               │
│ Latency            │ Lower          │ Slightly higher   │
│ Health check       │ TCP handshake  │ HTTP GET /health  │
│ Cost               │ Cheaper        │ Slightly more     │
│ Protocol support   │ TCP, UDP, TLS  │ HTTP, HTTPS       │
└─────────────────────────────────────────────────────────┘
```

**Your SAP BTP context:** SAP BTP uses L7 load balancers for routing between microservices — because it needs path-based routing (`/btp/v1/subaccounts` → SubAccount Service, `/btp/v1/environments` → Environment Service). Your Terraform Provider calls these paths, routing through an L7 LB.

### Load Balancing Algorithms

**Round Robin** (most common, simplest)
```
Server 1 → Server 2 → Server 3 → Server 1 → Server 2 → ...

Request 1  → Server 1
Request 2  → Server 2
Request 3  → Server 3
Request 4  → Server 1 (cycle repeats)

Pro: Simple, even distribution (assuming equal request cost)
Con: Doesn't account for server load — if Server 2 has a heavy request
     and Server 1 is idle, you might still send to overloaded Server 2
```

**Weighted Round Robin**
```
Server 1: weight 3 (powerful server)
Server 2: weight 1 (weaker server)

Requests go: S1, S1, S1, S2, S1, S1, S1, S2...
3x more traffic to Server 1

Use when: you're migrating from old to new servers (slowly shift traffic)
  Old server: weight 9
  New server: weight 1  → 10% of traffic goes to new server
  Monitor: if new server works fine, increase weight gradually
```

**Least Connections**
```
Route to whichever server has fewest active connections

Server 1: 100 active connections
Server 2: 50 active connections  ← new request goes here
Server 3: 75 active connections

Pro: More intelligent — accounts for server load
Con: Requires LB to track connection count per server
Use when: requests have variable processing time (some fast, some slow)
          → Common for WebSocket connections, long-lived connections
```

**IP Hash**
```
hash(client_ip) % number_of_servers → route to that server

Client IP 192.168.1.1 → hash → Server 2 (always)
Client IP 10.0.0.5    → hash → Server 1 (always)

Same client always goes to same server (sticky sessions)
Pro: Useful if server maintains some local state per user
Con: Can cause uneven distribution if certain IPs are very active
     If a server dies: all its clients get reassigned (cache loss)
     
Better alternative: session state in Redis (then you don't need sticky sessions)
```

**Random**
```
Pick a random server for each request.
Surprisingly effective in practice — with many servers, 
the distribution converges to even over time (law of large numbers).
```

### Health Checks

Load balancers must know which servers are alive. They do this with health checks.

```
Active Health Check:
  LB → sends GET /health every 10 seconds to each server
  
  If 200 OK: server is healthy → keep sending traffic ✓
  If 5xx/timeout for 3 consecutive checks: server is unhealthy
    → Stop sending traffic
    → Try again every 30 seconds
    → If healthy again → resume sending traffic

Passive Health Check:
  LB watches actual traffic
  If Server 2 returns errors for 10% of requests → mark unhealthy
  Pro: uses real traffic as the signal
  Con: users already saw the errors before server was removed
```

### SSL Termination at Load Balancer

```
Without SSL termination:
  Browser ←—HTTPS—→ LB ←—HTTPS—→ Server
  Each server must: decrypt HTTPS, process, encrypt response
  Each server needs SSL certificate and does crypto work

With SSL termination at LB:
  Browser ←—HTTPS—→ LB ←—HTTP—→ Servers (inside trusted network)
  LB decrypts HTTPS once, sends plain HTTP to servers
  Servers: no crypto work needed, much simpler
  Only LB needs the SSL certificate (managed once, not per-server)
  
  Pro: Servers are simpler, cheaper, less CPU for crypto
  Con: Traffic between LB and servers is unencrypted
       OK if they're in the same VPC/private network
       Not OK if servers are in different networks/clouds
```

---

## How These 3 Systems Work Together

Let's trace a full request to understand how DNS, CDN, and LB collaborate:

```
User opens sap-btp-portal.sap.com

1. DNS: "sap-btp-portal.sap.com" → 104.18.X.X (Cloudflare CDN IP)
   User's request goes to Cloudflare edge in nearest city

2. CDN (Cloudflare): 
   Is this a static asset? 
   → If yes (CSS, JS, images): serve from CDN cache → 5ms response ✓
   → If no (API call to get BTP subaccounts): forward to origin

3. Origin Load Balancer (ALB):
   Request arrives from CDN
   Checks URL path: /api/v1/subaccounts
   Routes to: SubAccount Service cluster
   Uses: least connections algorithm
   Sends to: Server 3 (currently least loaded)
   
4. Server 3 (Go service):
   Validates JWT token
   Reads from DB/cache
   Returns response

5. Response travels back:
   Server 3 → ALB → CDN → User
   CDN: Should I cache this? 
   → Check Cache-Control header
   → "private, no-cache" → don't cache (user-specific data)
   → Returns to user

Total time: ~100-200ms
```

---

## Summary — 5 Things to Remember

```
1. DNS translates domain names to IP addresses.
   TTL controls how long caches keep the answer.
   Lower TTL before planned changes, raise it back after.

2. CDN caches your static content at edge nodes worldwide.
   Use it for CSS, JS, images, anything that's the same for all users.
   Cache-bust by putting content hash in filenames.

3. Load Balancers distribute requests across your server fleet.
   L4 = fast, TCP-level, simple routing.
   L7 = HTTP-aware, path-based routing, SSL termination.

4. Health checks make your system self-healing.
   Failed server → automatically removed from rotation.
   Recovered server → automatically added back.

5. These three systems are often invisible but they determine:
   Your latency (CDN → edge proximity)
   Your availability (LB health checks → auto-failover)
   Your scalability (LB → distribute to unlimited servers)
```

---

## Quick Quiz

1. Your team changed the DNS record for `api.yourapp.com` from old IP to new IP, but some users are still hitting the old server an hour later. What's the cause and how do you prevent it next time?

2. You have a microservices architecture with these routes:
   - `/api/auth/*` → Auth Service
   - `/api/products/*` → Product Service
   - `/static/*` → S3 bucket
   
   Do you need L4 or L7 load balancer? Why?

3. Your load balancer uses Round Robin across 3 servers. Server 2 starts taking 30 seconds to process requests (stuck on a slow DB query) while Server 1 and 3 are fast. What happens to users? What would Least Connections do differently?

*(Answers:
1. Old TTL was too high — DNS resolvers cached the old record. Fix: lower TTL to 60s a few hours before future changes, then change record.
2. L7 — you need path-based routing (different paths → different services) which L4 cannot do.
3. Round Robin: 1/3 of users get sent to Server 2 and wait 30 seconds. LB doesn't know Server 2 is slow, just keeps sending to it. Least Connections: Server 2 accumulates connections (each request staying for 30s), so LB sees Server 2 has 100 connections vs Server 1/3 with 10 — stops sending to Server 2 automatically.)*

---

*Next: [Weekend Practice]
