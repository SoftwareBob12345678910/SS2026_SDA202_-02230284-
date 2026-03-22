# Practical 1 — Critical Analysis: Designing a URL Shortener (Chapter 8)

---

## 1. Problem Statement Analysis

The chapter tackles one of the most classic system design problems asked in technical interviews — **building a URL shortening service** similar to TinyURL or bit.ly.

The core idea sounds simple: take a long, ugly URL and convert it into a short, clean one. But when you dig deeper, the problem becomes surprisingly complex. Here's what the system actually needs to handle:

**The Three Core Requirements:**
- **URL Shortening** — A user gives a long URL, the system returns a short one
- **URL Redirecting** — When someone clicks the short URL, they get sent to the original long URL
- **Non-Functional Requirements** — The system must be highly available, scalable, and fault-tolerant

**The Scale of the Problem (Back-of-Envelope Estimation):**

The author sets up the following assumptions through a mock interview dialogue:
- 100 million URLs generated per day
- Read-to-write ratio of 10:1 (meaning people click links far more than they create them)
- Service runs for 10 years

This translates to:
- ~1,160 write operations per second
- ~11,600 read operations per second
- 365 billion total records over 10 years
- ~365 TB of storage required

 **Why this matters:** These numbers immediately tell us this cannot be a simple single-server application. We're dealing with a distributed, high-traffic system from day one.

---

## 2. Solutions Given by the Author

The author breaks the solution into four structured steps, which is itself a framework worth remembering for any system design interview.

---

### Step 1 — Ask Clarifying Questions First

Before writing a single line of design, the author emphasizes asking the right questions:
- What's the expected traffic?
- How short should the URL be?
- What characters are allowed?
- Can URLs be deleted or updated?

This is a critical real-world skill. **Designing without clarifying is like building a house without asking how many rooms are needed.**

---

### Step 2 — High-Level Design

**The API Design (Two Simple Endpoints):**

The entire system only needs two REST API endpoints:

```
POST /api/v1/data/shorten
  - Input: { longUrl: "https://very-long-url.com/..." }
  - Output: shortURL

GET /api/v1/{shortUrl}
  - Output: Redirects to longURL
```

**301 vs 302 Redirect — An Important Trade-off:**

This is one of the most insightful parts of the chapter. When a user visits a short URL, how should the server respond?

| Redirect Type | Behavior | Best For |
|---|---|---|
| **301 (Permanent)** | Browser caches the result. Future visits skip the shortener server entirely. | Reducing server load |
| **302 (Temporary)** | Every visit goes through the shortener server. | Tracking analytics (click counts, sources) |

 This is a **design trade-off**, not a right-or-wrong answer. Real systems must choose based on their priorities.

---

### Step 3 — Deep Dive Design

This is the technical heart of the chapter, covering three major areas:

**A) Data Model**

Instead of storing everything in memory (a hash table), which is expensive and volatile, the author recommends a simple **relational database table** with three columns:

```
url_table
-----------
id          (Primary Key, Auto Increment)
shortURL    (varchar)
longURL     (varchar)
```

This is simple, scalable, and easy to query.

---

**B) The Hash Function — Two Approaches**

The short URL needs a unique 7-character identifier (called a `hashValue`). Why 7 characters? Because using characters `[0-9, a-z, A-Z]` gives us 62 possible characters, and:

```
62^7 = ~3.5 trillion  ✅  (more than enough for 365 billion URLs)
62^6 = ~56 billion    ❌  (not enough)
```

The author explores **two approaches** to generate this hashValue:

**Approach 1: Hash + Collision Resolution**
- Apply a well-known hash function (CRC32, MD5, SHA-1) to the long URL
- Take the first 7 characters of the result
- **Problem:** Different long URLs might produce the same 7-character prefix (collision)
- **Solution:** If a collision occurs, append a predefined string to the URL and re-hash, repeating until no collision is found
- **Optimization:** Use **Bloom Filters** to quickly check if a shortURL already exists without hitting the database every time

**Approach 2: Base 62 Conversion (Preferred)**
- Assign each new URL a unique auto-incremented ID (e.g., `2009215674938`)
- Convert that ID from base 10 to base 62 using characters `[0-9, a-z, A-Z]`
- This naturally produces the short URL (e.g., `zn9edcu`)
- **No collision is possible** because every ID is unique by definition

**Comparison Table:**

| Feature | Hash + Collision Resolution | Base 62 Conversion |
|---|---|---|
| Short URL length | Fixed | Grows with ID size |
| Collision | Possible, must be handled | Impossible |
| Needs unique ID generator | No | Yes |
| Security | Better (not predictable) | IDs are sequential — guessable |

---

**C) URL Shortening Flow (Step-by-Step)**

```
1. User submits a long URL
2. Check if it already exists in the database
   → If YES: return the existing shortURL
   → If NO: continue
3. Generate a new unique ID using a distributed ID generator
4. Convert the ID to base 62 → this becomes the shortURL
5. Save (ID, shortURL, longURL) to the database
6. Return shortURL to the user
```

---

**D) URL Redirecting Flow (Step-by-Step)**

```
1. User clicks a short URL
2. Request hits the Load Balancer
3. Load Balancer forwards to a Web Server
4. Web Server checks the Cache (fast memory lookup)
   → If FOUND in cache: return longURL immediately
   → If NOT FOUND: query the database
5. Return longURL to the user (redirect)
```

 **Why caching matters here:** The system has 10x more reads than writes. Storing frequently accessed short URLs in a cache (like Redis) dramatically reduces database load and speeds up responses.

---

### Step 4 — Additional Considerations

The author wraps up with production-level concerns that are important in real systems:

- **Rate Limiting** — Prevent abuse by blocking users who make too many requests
- **Web Server Scaling** — Since servers are stateless, you can easily add/remove them horizontally
- **Database Scaling** — Use replication (for read performance) and sharding (for distributing data)
- **Analytics** — Track who clicked what and when using the 302 redirect approach
- **Availability & Consistency** — Referenced from earlier chapters as foundational concepts

---

## 3. My Understanding, Confusions & Topics to Explore Further

### What I Understood Clearly

The author does an excellent job building from simple to complex. The mock interview format at the beginning is genuinely useful — it taught me that in a real interview, **requirements clarification is not optional, it's the first step**.

The 301 vs 302 redirect distinction was something I had never thought about deeply before. On the surface they seem like minor HTTP details, but they have a real architectural impact — one saves money, the other saves data. That's a meaningful trade-off.

The Base 62 approach clicked very well for me. The math is simple and elegant: if you have a unique integer ID, you can always reverse-engineer a unique short string from it. No collision is theoretically possible. It's clean.

I also appreciated the caching strategy in the redirecting flow. Since the system is read-heavy (10:1 ratio), not caching would mean the database gets hammered on every single click. Caching the most popular short URLs is not just an optimization — it's practically a necessity at this scale.

---

### What Confused Me or Needs More Thought

**1. The Distributed Unique ID Generator**

The author mentions that generating globally unique IDs in a distributed environment is "challenging" and points to Chapter 7. This is actually a deeply complex problem. What happens if two servers try to generate an ID at the same exact millisecond? How do you guarantee uniqueness without a single centralized counter (which becomes a bottleneck)? Solutions like Twitter's **Snowflake ID** or **UUID** generation exist, but the chapter glosses over this. I'd like to explore this more.

**2. Bloom Filters**

The author briefly mentions Bloom Filters as a way to avoid expensive database lookups during collision resolution. I understand the concept at a surface level — it's a probabilistic data structure that can quickly tell you "this definitely does NOT exist" or "this probably exists." But the false-positive behavior and how it integrates with a production database is something I want to study deeper.

**3. Security of Base 62 Approach**

The comparison table mentions that Base 62 URLs are guessable because IDs increment by 1. This is a real security concern — if I shorten a private document and share it, someone could potentially guess adjacent IDs and access other people's links. The chapter acknowledges this but doesn't offer a solution. **How do real systems handle this?**

**4. No Delete/Update Support**

The chapter simplifies the problem by saying URLs cannot be deleted or updated. But in the real world (think Bitly), users can absolutely delete or edit their links. What does that do to the database design? What about cached entries for deleted URLs? This wasn't addressed.

---

### Topics I Want to Explore Further

- **Snowflake ID / Distributed ID Generation** — Deep dive into Chapter 7 of the same book
- **Bloom Filters** — Data structure, false positive rate, real-world use in databases like Cassandra
- **Consistent Hashing** — For distributing the database shards
- **Redis Caching Strategies** — LRU eviction, TTL settings, cache invalidation
- **Rate Limiting Algorithms** — Token bucket, leaky bucket, sliding window
- **CDN Integration** — Could static redirects be served from edge servers?
- **URL Security** — Preventing phishing/malicious links in shorteners

---

## 4. Brief Summary of Critical Analysis

Chapter 8 presents a well-structured, interview-ready framework for designing a URL shortener. The problem is deceptively simple — shorten a URL — but the author successfully reveals the layers of complexity hiding underneath: scale estimation, hashing strategies, redirect behavior, caching, and distributed ID generation.

**The strongest parts of the chapter** are the 301 vs 302 redirect trade-off discussion, the clear comparison between Hash+Collision and Base 62 approaches, and the layered architecture showing how caching sits between the web server and database to handle the read-heavy workload.

**The weaker parts** are the hand-waving around the distributed ID generator and the complete absence of URL deletion/update handling, both of which are real-world necessities. The security implication of sequential Base 62 IDs is mentioned but left unsolved.

Overall, this chapter teaches not just how to design a URL shortener, but how to think about any large-scale system: clarify requirements, estimate scale, design the API, choose data structures carefully, handle edge cases, and always think about trade-offs rather than absolute answers. That meta-skill is arguably more valuable than the specific solution itself.

---

Prepared by: [Karma Namgay Dorji]

Module: System Design
Reference: "System Design Interview" — Chapter 8: Design a URL Shortener