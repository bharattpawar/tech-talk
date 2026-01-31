# Discord's Database Evolution: "MongoDB se Trillions Tak"
## A Tech Talk Script on Scaling Message Storage at Internet Scale

**Speakers:** Rohit & Aditya  
**Duration:** 90 minutes  
**Format:** Conversational Hindi-English, technical + storytelling  
**Target Audience:** Full-stack engineers, system designers, competitive programmers

---

## INTRO (0:00 – 7:00)

**Rohit:**  
Aditya, sunao. Tum Discord par har din sab ko messages bhejte ho.  
Kya kabhi sochao... ye sab messages KAHAN store hote hain?

**Aditya:**  
Bhai, ek database mein obviously.

**Rohit:**  
Haan, par KAUN SA database?  
Ek simple question, par answer bohot complicated hai.

**Aditya:**  
Kyun?

**Rohit:**  
Kyun ki Discord ko 2026 mein ROZ 4 TRILLION messages store karne padh rahe hain.  
TRILLION. CHAR LAKH CRORE.

**Aditya:**  
Okay that's... insane.

**Rohit:**  
Aur ye journey? MongoDB se shuru hua.  
Phir Cassandra.  
Phir ScyllaDB.

Har migration ek new problem solve karta hai.  
Aur har problem, new issues create karta hai.

**Aditya:**  
Toh aaj ka topic?

**Rohit:**  
"How Discord Scaled From Hundreds of Millions to Trillions of Messages."  
Ya, simpler mein:  
**"Jab Database ko Millions se Trillions Tak Lana Padhe."**

**Aditya:**  
Chalo chalte hain.

---

## SEGMENT 1: THE BEGINNING – MONGODB ERA (7:00 – 18:00)

**Rohit:**  
2015. Discord abhi naya tha.  
Chhota startup, hundreds of users.

Kya use kiya message storage ke liye?

**Aditya:**  
MongoDB?

**Rohit:**  
Bilkul.

**Why MongoDB?**

**Rohit:**  
Simple reasons:
1. **Fast to deploy** – Ek command aur deployment ho gaya
2. **Schemaless** – Structure decide karao, koi puchega nahi
3. **Good for startups** – Perfect har fast-growing company ke liye

**Aditya:**  
Toh kya problem tha?

**Rohit:**  
Bas growth tha.

**The Numbers:**

**Rohit:**  
November 2015. Discord ne 100 MILLION messages store kar diye.

Ab sunao... ek message ka size kya hota hai?  
Roughly... 500 bytes to 1 KB.

100 million × 500 bytes = 50 GB.

**Aditya:**  
Toh 50 GB RAM mein fit ho jaata hai?

**Rohit:**  
Fit hota... agar sirf data hota.  
Par MongoDB ka ek rule hota hai:  
**"Data aur uske indexes RAM mein rehne chahiye."**

MongoDB ka compound index? On channel_id aur created_at.

**Why Compound Index?**

**Rohit:**  
Socho.  
Tum ek discord server mein jao.  
Tum chahte ho: "Mujhe last 50 messages dekha do."

Database ko 2 cheezon ki zaroorat hai:
1. **Kaun sa channel?** (channel_id)
2. **Sabse latest messages?** (created_at)

Compound index iska kaam tha.

Lekin 100 million messages pe?  
Index ka size = additional 20-30 GB.

Total = 70-80 GB.

**Aditya:**  
Ab 16 GB RAM mein fit nahi hoga.

**Rohit:**  
Bilkul.  
MongoDB ko disk se read karna padne laga.

**What Happens Then?**

**Rohit:**  
Disk speed = RAM speed ka 1000x slower.

**Example:**
- RAM read = 1 microsecond
- Disk read = 1 millisecond

Ek latency spike ho jaati hai.

**Aditya:**  
Toh user ko kya dikh raha tha?

**Rohit:**  
Messages load ho rahe the, par intermittently slow.  
Kabhi instant, kabhi 2-3 seconds lag.

Aur ye unpredictable tha.  
Sometimes fast, sometimes slow.  
Engineering team ko samajh nahi aa raha... kyun?

**The MongoDB Limitation:**

**Rohit:**  
Here's the thing:  
MongoDB handles writes well.  
But it's fundamentally single-node optimized.

Sharding karte ho?  
Complexity explodes.  
Consistency hard ho jaati hai.

**Aditya:**  
So Discord ko MongoDB chhod dena pada?

**Rohit:**  
Haan.  
By 2017, they were storing BILLIONS of messages.

MongoDB tha hi nahi suitable.

---

## SEGMENT 2: THE MIGRATION – CASSANDRA ERA (18:00 – 38:00)

**Rohit:**  
2017. Discord engineers ko ek naya database chosen karna tha.

Kaunsa?

**Aditya:**  
Apache Cassandra.

**Rohit:**  
Bilkul.

**Why Cassandra?**

**Rohit:**  
Cassandra ek distributed database hai.  
Isme built-in support hai:
1. **Horizontal scaling** – Node add karo, data automatically distribute hoga
2. **High write throughput** – Millions of writes per second
3. **Data replication** – Same data multiple nodes pe
4. **Eventual consistency** – Fast writes, later consistency

**Aditya:**  
Toh basically... perfect for chat apps?

**Rohit:**  
Bilkul perfect.

**Discord's Cassandra Setup:**

**Rohit:**  
Table structure:

```
CREATE TABLE messages (
  channel_id bigint,      -- Kaun sa channel
  bucket int,             -- Time window (hour/day/week)
  message_id bigint,      -- Unique message ID (Snowflake)
  user_id bigint,
  content text,
  attachments list,
  reactions map,
  created_at timestamp,
  ...
);

PRIMARY KEY ((channel_id, bucket), message_id DESC);
```

**Aditya:**  
Compound partition key?

**Rohit:**  
Haan.  
(channel_id, bucket) = partition key.  
message_id = clustering key (sorted descending).

**Why This Design?**

**Rohit:**  
Socho. Tum ek message fetch karte ho:

```
SELECT * FROM messages 
WHERE channel_id = 12345 
AND bucket = 202601  -- January 2026
ORDER BY message_id DESC 
LIMIT 50;
```

Cassandra kya karega?
1. Go to partition (channel 12345, bucket 202601)
2. Messages already sorted DESC by message_id
3. Return top 50

**Speed?** < 1 millisecond.

**Aditya:**  
Brilliant.

**Rohit:**  
Brilliant, bilkul.

By 2017, Discord had 12 Cassandra nodes.  
billions of messages.

Latency = less than 5 milliseconds for reads, < 1 ms for writes.

**Aditya:**  
Perfect performance?

**Rohit:**  
Perfect tha... tab tak.

---

**FIVE YEARS PASS (38:00 – 45:00)**

**Rohit:**  
2022.

Discord ab 800 million users.  
Daily 5 billion messages.

Message count? TRILLIONS.

Cassandra nodes? 12 se jump ho gaya... 177.

**Aditya:**  
That's crazy growth.

**Rohit:**  
Haan. Lekin with growth came... PROBLEMS.

---

## SEGMENT 3: THE CASSANDRA PAIN POINTS (45:00 – 62:00)

**Red Flag #1: Hot Partitions**

**Rohit:**  
Discord mein millions of channels.

Most channels? 100 users, casual messages.

But some channels?  
Trending servers, gaming communities, Pokemon Go communities...

**Popular Channel Example:**

**Rohit:**  
Channel A = 5 million concurrent users.  
Say, World Cup Final mein, live commentary.

Har second, 50,000 messages.

**What Happens?**

**Rohit:**  
Partition key: (channel_id, bucket)

Channel ID = same.  
Bucket = same time window.

So all 50,000 messages go to... ONE partition.  
Which might live on... ONE node (or replica set).

That node gets HAMMERED.

**Aditya:**  
Hot partition?

**Rohit:**  
Bilkul.

Node tries to serve traffic.  
CPU maxes out.  
Memory gets paged.  
Disk I/O increases.

**Latency skyrockets.**

Aur here's the bad part:  
Cassandra uses **quorum consistency**.  
Multiple replicas need to ack.

Agar ek replica slow hai, SABA slow ho jaata hai.

**Aditya:**  
So one hot partition = entire cluster slow?

**Rohit:**  
Exactly.

User experience = lag, timeouts, retries.

**Aditya:**  
How do you fix this?

**Rohit:**  
For Discord?  
Nahi fix kar sakte at Cassandra level.  
Architecture hi change karna padha.

(We'll come back to this.)

---

**Red Flag #2: Garbage Collection Hell**

**Rohit:**  
Cassandra Java mein likha hota hai.  
Java = JVM.  
JVM = needs garbage collection.

**What's GC?**

**Rohit:**  
JVM memory mein allocated.  
Jab memory full hota hai, JVM PAUSES.  
Sab threads stop.  
GC runs, unused memory clear karta hai.  
Phir threads resume.

**Kya problem hota hai?**

**Rohit:**  
Pause time = 10-50 milliseconds.  
Or even SECONDS in extreme cases.

Discord ka Cassandra cluster?  
177 nodes.  
Constant writes, constant reads.

GC pause hota, ek node stop hota.  
Quorum consistency mein, request timeout.

**Aditya:**  
Network timeouts?

**Rohit:**  
Haan. User ko message send karne ka request timeout.  
Retry hota.  
Retry bhi slow, kyun ki nodes GC mein fas gaye.

**The Knobs Problem:**

**Rohit:**  
JVM engineers ko JVM tune karna padta tha:

```
-XX:+UseG1GC                    # GC algorithm
-XX:MaxGCPauseMillis=200        # Max pause time
-XX:G1HeapRegionSize=16m        # Heap region size
-XX:+ParallelRefProcEnabled     # Reference processing
-XX:+UnlockExperimentalVMOptions
-XX:G1NewCollectionHeuristicWeight=60
-XX:G1SummarizeRSetStatsPeriod=86400
```

**Aditya:**  
Woah. How many flags?

**Rohit:**  
100+.

On-call engineer, weekend pe alert aata hai:  
"p99 latency spike."

Root cause = wrong GC setting.  
Change ek flag.  
Re-deploy.  
Wait 2 hours.  
Hope it works.

**Aditya:**  
That sounds like a nightmare.

**Rohit:**  
It WAS.

---

**Red Flag #3: Compaction Backlog**

**Rohit:**  
Cassandra mein ek process hoti: SSTable compaction.

**What's Compaction?**

**Rohit:**  
Cassandra writes data ka log append karta hai.  
Overtime, multiple log files (SSTables) accumulate.

Pढ़ने ke liye:
- Multiple files scan karne padh sakte hain
- Slower

Compaction: merge karke single file banana.

**Aditya:**  
Sounds efficient?

**Rohit:**  
Haan, but EXPENSIVE.

Compaction process CPU aur disk I/O use karte hain.  
Agar production pe heavy writes ho rahe, compaction lag jaate hain.

**Result?**  
Many SSTables on disk.  
Reads become... expensive.  
Latency increases.

Discord ke 177 nodes pe?  
Compaction often fell behind.  
Team had to manually babysit nodes, take them out, let them catch up.

**Aditya:**  
So... constant firefighting?

**Rohit:**  
Constant.

---

**Red Flag #4: The Tombstone Problem**

**Rohit:**  
Ek specific issue jishe Discord ne face kiya.

Cassandra deletes ko "eventually consistent" way mein handle karta hai.

Jab tum ek message delete karte ho, Cassandra doesn't immediately remove it.  
Instead, it marks a "tombstone."

**Why?**

**Rohit:**  
Distributed system mein, node A pe delete hota, node B ko abhi pta nahi.

Node B ko message mil sakta hai.  
Reconcile karate time, tombstone prevent karti data from coming back.

**The Problem:**

**Rohit:**  
Discord schema mein, 16 columns the average.  
But most messages sirf 3-4 values set karte the.

Rest? Null.

Cassandra mein, null value ko bhi write karte ho.  
Aur delete/update = tombstone.

Agar 12 messages te null values, 12 tombstones create hote the.

**At Scale:**

**Rohit:**  
Popular channel, millions of messages.  
Millions of tombstones.

User ek channel open karta, past messages fetch karate.  
Database ko millions of tombstones scan karna padta.

Latency = 100+ milliseconds.

**Aditya:**  
How'd they fix it?

**Rohit:**  
Simple:  
"Only write non-null values."

If column is null, don't write it at all.

Tombstone problem solved.

---

## SEGMENT 4: THE DECISION – TIME FOR SCYLLADB (62:00 – 72:00)

**Rohit:**  
By 2022, Discord engineers ko clear tha:  
**Cassandra ka maintenance overhead unsustainable tha.**

177 nodes.  
Constant GC tuning.  
Hot partitions killing p99 latencies.  
Compaction backlogs.

**What were the options?**

**Rohit:**  
1. **Scale Cassandra more** – Add more nodes (more ops burden)
2. **Rewrite on new DB** – Risky, downtime possible
3. **Find Cassandra alternative** – Something similar, but better

Discord chose #3.

**Enter ScyllaDB:**

**Rohit:**  
ScyllaDB. Cassandra-compatible database.  
Written in C++.

Same API as Cassandra.  
But fundamentally different internals.

**Why C++?**

**Rohit:**  
1. **No garbage collection** – C++ manual memory management
2. **Shard-per-core architecture** – Each CPU core gets its own data + thread
3. **Lock-free algorithms** – Minimal contention
4. **Direct memory access** – No JVM abstraction overhead

**Aditya:**  
So basically... same API, better performance?

**Rohit:**  
Better performance, better predictability, way less ops burden.

---

## SEGMENT 5: THE MIGRATION STRATEGY (72:00 – 82:00)

**Rohit:**  
Migrating trillions of messages = NOT simple.

Risk: downtime, data loss, consistency issues.

Discord ka approach?

**Step 1: Dual-Write Phase**

**Rohit:**  
New messages start getting written to BOTH databases:
- Cassandra (old)
- ScyllaDB (new)

Code logic:
```
Write message to Cassandra
Write message to ScyllaDB
If both succeed → return success
If either fails → retry
```

Duration: 2-3 months.  
Purpose: Warm up ScyllaDB cluster.

**Step 2: Data Migration Tool**

**Rohit:**  
Trillions of old messages ko ScyllaDB mein migrate karna.

Tool written in: **Rust**.

**Why Rust?**

**Rohit:**  
1. No garbage collection (like C++)
2. Concurrency without data races
3. Can migrate fast without impacting production

Tool ka logic:
```
Scan Cassandra partitions
Batch read (say 10,000 messages)
Write to ScyllaDB
Verify (checksum match)
Continue next batch
```

**Aditya:**  
How long did migration take?

**Rohit:**  
9 days.

4 TRILLION messages.  
9 days.

Matlab... 460 million messages per day.  
5,000 messages per second transfer.

**Aditya:**  
While production running?

**Rohit:**  
Bilkul. Zero downtime.

**Step 3: Read Validation**

**Rohit:**  
Small percentage of reads (say 1%) sent to BOTH databases.

```
Get result from Cassandra
Get result from ScyllaDB
Compare
If mismatch → log, alert, fix
```

If all validations pass, increment percentage.  
1% → 10% → 50% → 100%.

**Aditya:**  
What if mismatch happens?

**Rohit:**  
Debug, find root cause, fix code/data, resume.

**Step 4: Full Cutover**

**Rohit:**  
Once 100% reads validated:
1. Stop dual-writes
2. Switch all writes to ScyllaDB
3. Monitor latencies, errors
4. Keep Cassandra as fallback for 1-2 months
5. Decommission Cassandra

---

## SEGMENT 6: HANDLING HOT PARTITIONS – RUST DATA SERVICE (82:00 – 95:00)

**Rohit:**  
Remember hot partition problem?

ScyllaDB solves GC issue, but not hot partition issue.

Architecture se solve karna padta hai.

**Discord's Solution: Data Service Layer**

**Rohit:**  
Between API servers aur ScyllaDB, ek NEW service layer:

```
API Servers
    ↓
Data Service (Rust)
    ↓
ScyllaDB
```

**What does Data Service do?**

**Rohit:**  
1. **Request coalescing**
2. **Consistent hashing**
3. **Intelligent routing**

**Request Coalescing:**

**Rohit:**  
Scenario: World Cup final. 5M users.

All requesting: "Get last 50 messages from channel X."

Without coalescing:  
5M separate database queries.  
Database crushed.

With coalescing:  
Data service receives 5M requests.  
Sees: "All requesting same data."

Hits database ONCE.  
Caches result.  
Returns to all 5M users.

**Aditya:**  
Basically a read-through cache?

**Rohit:**  
Exactly.

**Consistent Hashing:**

**Rohit:**  
Data service instances: say 100.

Request comes with routing key (channel_id).

```
hash(channel_id) % 100 = instance X
Send request to instance X only
```

Benefit: Same channel always goes to same data service instance.  
That instance's cache is warm.  
Cache hit rate = high.

**Result:**  
Hot partitions STILL exist in database.  
But requests are coalesced.  
Database load = much lower.

**Aditya:**  
Genius.

**Rohit:**  
Bilkul genius.

---

## SEGMENT 7: THE RESULTS (95:00 – 108:00)

**Node Reduction:**

**Rohit:**  
Before ScyllaDB: 177 Cassandra nodes  
After ScyllaDB: 72 nodes

**59% reduction.**

Not sirf nodes decrease, disk utilization bhi:

```
Cassandra: 177 nodes × 4TB = 708 TB
ScyllaDB: 72 nodes × 9TB = 648 TB (better compression)
```

**Latency Improvements:**

**Rohit:**  
Here are real numbers Discord published:

| Metric | Cassandra | ScyllaDB | Improvement |
|--------|-----------|----------|-------------|
| **p50 Read Latency** | 2-5 ms | 1-2 ms | 50-60% |
| **p99 Read Latency** | 40-125 ms | 15 ms | **88% reduction** |
| **p50 Write Latency** | 1-2 ms | 0.5-1 ms | 50% |
| **p99 Write Latency** | 5-70 ms | 5 ms | **93% reduction** |
| **Historical Reads (p99)** | 5-200 ms (unpredictable) | 5 ms (consistent) | **Predictable** |

**Aditya:**  
p99 reads improved 88%?

**Rohit:**  
Haan.

**What does p99 mean?**

**Rohit:**  
99% of reads complete in 15 ms.  
Only 1% are slower.

Before: 99% in 125 ms. 1% in 200+ ms.  
After: 99% in 15 ms. 1% in 20-30 ms.

**Aditya:**  
That's... huge for user experience.

**Rohit:**  
Massive.

User opens Discord, loads message history.  
On Cassandra: sometimes instant, sometimes 2-3 second delay.  
On ScyllaDB: always instant.

**Operational Burden:**

**Rohit:**  
Before ScyllaDB:
- GC tuning = 40 hours per month
- Compaction babysitting = 30 hours per month
- Hot partition firefighting = 50 hours per month
- Total: 120 hours per month per engineer

After ScyllaDB:
- No GC tuning
- Minimal compaction intervention
- Data service handles partitions
- Total: 10 hours per month per engineer

**Aditya:**  
90% reduction in ops burden?

**Rohit:**  
Bilkul.

---

## SEGMENT 8: WHY THIS MATTERS (108:00 – 118:00)

**Aditya:**  
Okay, so... why should we care?

**Rohit:**  
Niche reasons:

**1. Database Selection Matters**

**Rohit:**  
Ek wrong database choice = years of pain.

Discord chose right (Cassandra for its time).  
But as scale changed, needs changed.

Moral: Monitor your database health.  
If unpredictable latencies rising, architecture change shayad zaroori.

**2. Language Choice Matters**

**Rohit:**  
C++ vs Java = 10x latency difference in some cases.

For real-time systems (chat, gaming, finance):  
Language choice = critical.

**3. Predictability > Peak Performance**

**Rohit:**  
Cassandra: peak throughput = high, but p99 latency = unpredictable.  
ScyllaDB: slightly lower peak, but p99 = consistent.

Users prefer consistent, slightly slower over unpredictable fast.

Moral: Tail latency matters.

**4. Architecture Shields Database**

**Rohit:**  
Hot partitions problem unsolvable at database level.  
Solution: Data service layer.

Lesson: Sometimes add application layer logic to shield database.

**5. Migration Strategy Matters**

**Rohit:**  
Dual-write → Migrate → Validate → Cutover.

Zero downtime migration of trillions of records.

Not trivial, but possible with planning.

---

## SEGMENT 9: WHAT WOULD YOU CHOOSE? (118:00 – 128:00)

**Aditya:**  
So if we were building a chat app TODAY, what database?

**Rohit:**  
Depends on scale:

**Small scale (< 100M messages):**  
- PostgreSQL + Redis cache
- Simple, ACID guarantees, good performance

**Medium scale (100M - 1B messages):**  
- Cassandra or ScyllaDB
- Good distributed architecture

**Large scale (> 1B messages):**  
- ScyllaDB (if you want Cassandra compatibility)
- Or custom solution on cloud services
- Like Discord eventually built custom services on top

**Aditya:**  
Would you use MongoDB?

**Rohit:**  
For chat? No.

MongoDB good for:
- Document-heavy workloads
- Complex queries
- Smaller-scale systems

Chat is:
- High write throughput
- Simple queries
- Massive scale

MongoDB not ideal.

**Aditya:**  
What about NoSQL alternatives? Like DynamoDB?

**Rohit:**  
DynamoDB workable, but:
- Vendor lock-in (AWS)
- Hot partition problem still exists
- Costs can explode at scale

Open-source options (Cassandra/ScyllaDB) give more control.

---

## SEGMENT 10: LESSONS FOR YOUR PROJECTS (128:00 – 138:00)

**Rohit:**  
Tum sab student ho ya junior engineers.

Discord ka journey, tum apne chhote projects pe apply kar sakte ho:

**Lesson 1: Start Simple**

**Rohit:**  
PostgreSQL se start karo.  
SQLite even.

Jab performance issue ache hit, move to distributed system.

**Lesson 2: Monitor From Day 1**

**Rohit:**  
Latency tracking setup karo.  
p50, p99, p99.9 track karo.

Trends dekho.  
Early warning = better migration planning.

**Lesson 3: Architecture Over Performance Tuning**

**Rohit:**  
Agar system slow, first try: architectural fix.  
Cache add karo, load balancer add karo, service layer add karo.

Tuning come later.

**Lesson 4: Test Migration Strategy**

**Rohit:**  
Before production:
- Sandbox mein test karo
- Small data set pe
- Validate every step

**Lesson 5: Choose Boring Technology**

**Rohit:**  
Hype se matlab nahi.

PostgreSQL, Redis, Cassandra = boring, proven, reliable.

New DB? Shayad exciting, par risk high.

---

## CLOSING (138:00 – 145:00)

**Rohit:**  
Discord ne sochao.

2015 = 100 million messages.  
2026 = 4 TRILLION messages.

40,000x growth.

Ek database se doosre pe shift karna padha.  
Doosre se teesre pe.

Har choice: tradeoffs.
- MongoDB: simple, slow at scale
- Cassandra: scalable, ops burden
- ScyllaDB: scalable, predictable, but architectural changes needed (data service)

**Aditya:**  
So ye continuous process hota hai?

**Rohit:**  
Haan.

Companies ka database ka evolution never stops.

Netflix, Uber, Airbnb, sab ke different journeys.

But common theme:
**"As data grows, simplicity trades off for scalability."**

**Aditya:**  
Kyun?

**Rohit:**  
Kyun ki distributed systems hard hote hain.

Consistency, availability, partition tolerance.  
Pick 2, third compromised.

Simple systems = all three.  
But can't scale.

**Aditya:**  
Toh engineering = tradeoffs?

**Rohit:**  
Always.

**TOGETHER:**  
Pick your tradeoffs wisely.

---

## APPENDIX: QUICK REFERENCE

### Timeline
- **2015 (Nov)**: MongoDB → 100M messages, RAM overflow
- **2017**: Cassandra migration, 12 nodes, billions of messages
- **2022 (Jan)**: Cassandra → 177 nodes, trillions, ops burden high
- **2022-2023**: Migration planning, dual-write, data service build
- **2023 (June)**: ScyllaDB cutover complete, 72 nodes
- **2026**: ScyllaDB stable, handles 4 trillion+ messages

### Key Metrics
- **Growth rate**: 40,000x in 11 years
- **Node reduction**: 177 → 72 (59% reduction)
- **p99 read latency improvement**: 40-125ms → 15ms (**88% reduction**)
- **p99 write latency improvement**: 5-70ms → 5ms (**93% reduction**)
- **Ops burden reduction**: ~90%
- **Migration duration**: 9 days (zero downtime)
- **Migration tool**: Rust (5,000 msgs/sec transfer rate)

### Why Each Database Failed/Succeeded

| Database | Succeeded At | Failed At | Reason |
|----------|--------------|-----------|--------|
| **MongoDB** | 0-100M msgs | >100M msgs | Single-node optimized, complex sharding, index bloat |
| **Cassandra** | 100M-1T msgs | Ops, tail latency, hot partitions | Java GC, compaction lag, no partition isolation |
| **ScyllaDB** | 1T+ msgs | (none yet) | C++ arch, shard-per-core, no GC, + data service layer |

### Architectural Evolution
```
V1 (2015): MongoDB (single instance + cache)
   ↓
V2 (2017): Cassandra cluster (12 nodes, simple routing)
   ↓
V3 (2022): Cassandra cluster (177 nodes, GC hell, hot partitions)
   ↓
V4 (2023): ScyllaDB cluster (72 nodes) + Rust Data Service (request coalescing, consistent hashing)
   ↓
V5 (Future?): Custom cloud-native service? Further optimization?
```

### Key Technical Concepts
1. **Compound indexes**: (channel_id, created_at) for efficient message queries
2. **Partition keys**: (channel_id, bucket) to distribute load
3. **Hot partitions**: Popular channels → single node overload
4. **Garbage collection**: JVM pauses → unpredictable latency
5. **Tombstones**: Logical deletes, lingering in Cassandra
6. **SSTable compaction**: Merging write-ahead logs, CPU/IO intensive
7. **Eventual consistency**: Tradeoff for write throughput
8. **Quorum reads**: Multiple replicas must ack → P99 latency = slowest replica
9. **Shard-per-core**: C++ architecture for lock-free parallelism
10. **Request coalescing**: Collapse duplicate reads into single DB query

### Resources to Dive Deeper
- Discord blog: "How Discord Stores Trillions of Messages"
- ScyllaDB case study: Discord migration
- ByteByteGo article on Discord's architecture
- Architecture Patterns: Dual-write, strangler pattern, cache-aside

---

## Discussion Questions for Audience

1. **At what scale would YOU migrate from PostgreSQL to Cassandra?**
2. **Why is p99 latency more important than average latency for chat apps?**
3. **What are other scenarios where request coalescing helps? (Cache stampede, thundering herd)**
4. **Would you ever write your own database? When?**
5. **How would you monitor database health to catch issues early?**
6. **What's the difference between schemaful (Cassandra) and schemaless (MongoDB) trade-offs?**

---

## Notes for Presenters (Rohit & Aditya)

**Visual Aids to Prepare:**
1. Timeline graph: Messages stored over time (exponential curve)
2. Node count evolution: 1 → 12 → 177 → 72
3. Latency box plots: Cassandra vs ScyllaDB (p50, p99, p99.9)
4. Architecture diagrams: Each version (V1-V5)
5. Hot partition visualization: Traffic pattern on one node
6. GC pause diagram: Request timeline interrupted by pause
7. Compaction backlog impact: Read latency vs SSTable count

**Story Elements to Emphasize:**
- Human impact: Users experiencing lag, frustration
- Engineering pain: Oncall engineers' suffering during GC issues
- Scale perspective: 4 trillion = almost 600 messages per human on Earth
- Problem evolution: Each solution creates new problems
- Humility: "Even large companies struggle with scalability"

**Interactive Elements:**
- Live polling: "What DB would you choose?"
- Code walkthrough: Dual-write logic, migration validation
- Calculation exercise: "If you have 1B messages/day, how many nodes do you need?"
- Real quotes from Discord engineers (from blog/talks)

---

**END OF SCRIPT**
