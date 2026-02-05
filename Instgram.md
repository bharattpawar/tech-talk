# How Instagram Works: "Billions of Users, Milliseconds of Latency"

## A Deep Tech Talk Script on Instagram’s Architecture at Internet Scale

**Speakers:** Rohit & Aditya
**Duration:** ~90 minutes
**Format:** Conversational Hindi-English, technical + storytelling
**Target Audience:** Full‑stack engineers, system designers, backend developers

---

## INTRO (0:00 – 7:00)

**Rohit:**
Aditya, ek simple sawal. Tum Instagram roz kholte ho?

**Aditya:**
Roz? Bhai din mein 20 baar.

**Rohit:**
Aur har baar app instantly khul jata hai. Feed load ho jati hai. Reels chal jati hain.

Kabhi socha… Instagram itne BILLIONS users ko kaise handle karta hai?

**Aditya:**
Distributed systems magic?

**Rohit:**
Magic nahi. Engineering. Massive engineering.

Instagram ko handle karna padta hai:

* Billions of active users
* Millions of posts per minute
* Real‑time likes, comments, DMs
* Petabytes of photos and videos

Aaj hum breakdown karenge:

**"How Instagram Works Under the Hood."**

From request hitting your phone… to data centers… to AI ranking your feed.

---

## SEGMENT 1: HIGH‑LEVEL ARCHITECTURE (7:00 – 15:00)

**Rohit:**
Instagram ka architecture 5 major layers mein socho:

1. **Client Layer** – Mobile app / Web
2. **Edge Layer** – CDN + Load balancers
3. **API Layer** – Backend services
4. **Data Layer** – Databases + caches
5. **Storage + ML Layer** – Media + ranking

Flow simple lagta hai:

```
Phone → CDN → Load Balancer → API Services → Cache/DB → Response
```

But har arrow ke peeche complex systems hain.

**Aditya:**
Toh pehle client se shuru karein?

**Rohit:**
Exactly.

---

## SEGMENT 2: CLIENT + EDGE DELIVERY (15:00 – 25:00)

**Rohit:**
Jab tum Instagram kholte ho, app pehle nearest **CDN edge server** se connect karta hai.

Goal: latency minimize karna.

Static assets jaise:

* Images
* Videos
* Thumbnails
* JS bundles

CDN pe cached hote hain.

**Why CDN important?**

Agar har request US data center jaye, India user ko 200–300ms lag sakte hain.

CDN se:

* 20–40ms latency
* Less backend load

**Aditya:**
Dynamic data ka kya? Feed personalized hoti hai.

**Rohit:**
Dynamic requests load balancer se API servers pe jati hain.

Load balancers use karte hain:

* Anycast routing
* Health checks
* Traffic distribution

Taaki koi single server overload na ho.

---

## SEGMENT 3: API LAYER – MICROSERVICES (25:00 – 40:00)

**Rohit:**
Instagram monolith nahi hai. Ye **microservices architecture** use karta hai.

Different services:

* User service
* Feed service
* Media service
* Comment service
* Notification service
* Messaging service

Har service independently scale hoti hai.

**Aditya:**
Feed service sabse heavy hogi?

**Rohit:**
Bilkul.

Feed generation involves:

1. Fetch followed users
2. Pull recent posts
3. Rank using ML models
4. Filter spam/content rules
5. Return top N posts

Ye sab milliseconds mein hota hai.

Services communicate via:

* RPC / gRPC
* Internal service mesh

---

## SEGMENT 4: CACHING STRATEGY (40:00 – 52:00)

**Rohit:**
Agar Instagram har request DB se fetch kare… system collapse ho jayega.

Isliye heavy caching use hoti hai.

**Primary cache: Redis / Memcached**

Cached items:

* User sessions
* Feed fragments
* Popular posts
* Relationship graphs

**Cache‑Aside Pattern:**

```
Check cache → Miss → Fetch DB → Update cache
```

Hot users ke liye cache hit rate bohot high hota hai.

**Aditya:**
Cache invalidation kaise handle karte?

**Rohit:**
TTL + event‑driven invalidation.

Jab user new post karta hai:

* Followers ke feed cache invalidate
* Async jobs update ranking

---

## SEGMENT 5: DATABASE LAYER (52:00 – 65:00)

**Rohit:**
Instagram historically started with **PostgreSQL**.

Aur aaj bhi relational DB heavily use hoti hai.

But single DB enough nahi hoti.

They use:

* Sharded PostgreSQL clusters
* Distributed key‑value stores
* Graph storage for relationships

**Sharding strategy:**

Users ko shards mein divide kiya jata hai.

```
hash(user_id) → shard number
```

Benefit:

* Horizontal scaling
* Isolated failures

Replication use hoti hai for:

* Read scaling
* High availability

---

## SEGMENT 6: MEDIA STORAGE PIPELINE (65:00 – 75:00)

**Rohit:**
Photos/videos Instagram ka core hain.

Upload pipeline:

1. Client uploads to nearest ingest server
2. Media processing pipeline triggers
3. Multiple resolutions generate hote hain
4. Stored in distributed object storage
5. Replicated globally

Processing includes:

* Compression
* Format conversion
* Thumbnail generation

CDN then serves optimized versions.

**Aditya:**
Isliye reels fast load hoti hain?

**Rohit:**
Exactly. Pre‑processed + edge cached.

---

## SEGMENT 7: FEED RANKING + ML (75:00 – 85:00)

**Rohit:**
Instagram ka secret sauce: **ranking algorithms**.

Machine learning predicts:

* Probability of like
* Comment likelihood
* Watch time
* Engagement score

Each post gets a ranking score.

Top scoring posts appear first.

Pipeline:

1. Candidate generation
2. Feature extraction
3. ML scoring
4. Final ranking

All in near real‑time.

---

## SEGMENT 8: REAL‑TIME SYSTEMS (85:00 – 95:00)

**Rohit:**
Likes, comments, DMs need real‑time delivery.

They use:

* Event streaming systems
* Message queues
* Push notification pipelines

Events propagate asynchronously.

User experience feels instant, even if backend distributed hai.

---

## SEGMENT 8.5: DEEP DIVE — INSTAGRAM DMS (95:00 – 110:00)

**Rohit:**
Ab baat karte hain Instagram ke **Direct Messages (DMs)** ki — ek real, production-grade DM system kaise bana hota hai, aur Instagram specifically kya cheezein handle karta hai. Ye section DM internals ko step-by-step cover karta hai: connection protocol, routing, storage, sync, media, encryption, and safety features.

### 1) Connection + Transport

* **Persistent connections:** Instagram clients (mobile/web) usually maintain a persistent bidirectional connection with backend edge services. Common choices: **WebSockets** or lightweight publish/subscribe protocols (there are signals that Meta/Instagram use MQTT-like pub/sub patterns internally). Persistent sockets allow: real-time message delivery, typing indicators, presence, and read receipts with very low latency.

* **Multiplexing topics:** Each DM thread can map to a topic or channel (e.g., `/ig/dm/thread/<thread_id>`). Clients subscribe to only relevant topics so servers can push updates efficiently.

* **Edge gateways:** The mobile app connects to a geographically close edge gateway (TLS-terminated). Gateways authenticate the client, enforce rate limits, and route events into the messaging backbone.

### 2) Message flow (send → deliver → ack)

1. **Send:** Client sends a message over the socket to the nearest gateway with metadata: `sender_id`, `thread_id`, `client_msg_id`, `timestamp`, `payload` (text/media pointer), `sync_token`.
2. **Gateway validation:** Gateway verifies auth, rate limits, and forwards to the messaging service.
3. **Persistence enqueue:** Messaging service writes the message to a durable message store (append-only log / write-through DB) and assigns a server message id. This write ensures durability before acknowledging the sender.
4. **Fan-out:** The service fans out the message event to: connected recipients (via their gateway connections), push notification service (for offline recipients), and background workers for notifications/ML/safety checks.
5. **Delivery ack:** When a recipient's connected gateway acknowledges delivery, the sender can get a delivered/read ack. Reads/seen receipts are propagated similarly.

### 3) Storage model

* **Append-only message log:** Messages are stored in immutable append-only stores or logs for durability (think sharded message queues + cold storage).
* **Indexing for threads:** Secondary indices (by `thread_id`, `participant_id`, time) allow efficient retrieval of the last N messages for a thread.
* **Separate metadata store:** Thread metadata (participants, last-read watermark, pinned messages) lives in a low-latency DB (sharded Postgres / key-value store).
* **Retention & deletes:** Deletes may create tombstones; modern systems avoid unnecessary tombstones by writing only non-null values and using compacted storage. For privacy, end-users can delete from their view and deletion requests trigger background sync to remove or tombstone data across replicas.

### 4) Sync across devices + offline support

* **Sync tokens / watermarks:** Clients maintain a `sync_token` (last seen server sequence number). On reconnect, client asks: "give me everything after `sync_token`" — server returns delta.
* **Local caching:** Apps keep a local DB (SQLite/Realm) for chat history so UI is instant and to support offline compose. After reconnection, client reconciles local pending messages (using `client_msg_id`) with server-assigned ids.

### 5) Media in DMs (images, voice, videos, ephemeral content)

* **Pointer model:** Large media blobs are uploaded to object storage (S3-like) and the message payload stores a pointer (URL or content-hash) plus rendering metadata (resolutions, duration).
* **Ephemeral / disappearing media:** For stories or vanish mode, systems use short-lived object URLs and flag content for periodic deletion. Ephemeral content may be stored encrypted and lifecycle-managed more strictly.

### 6) Delivery guarantees & ordering

* **At-least-once delivery:** Messaging systems usually aim for at-least-once delivery with idempotency (client_msg_id) to deduplicate.
* **Ordering:** Per-thread sequence numbers preserve causal ordering for rendering (server sorts by server-assigned sequence id). Out-of-order arrivals from caches/gateways are reconciled by sequence number.

### 7) Read receipts, typing, presence

* **Low-traffic signals:** Typing indicators and presence are high-frequency low-size events — handled in-memory and not persisted long-term. They are often delivered with best-effort (no strong durability), reducing DB churn.

### 8) Push notifications & offline delivery

* **Push pipeline:** If recipient not connected, the messaging service publishes to a push-notification pipeline (APNs / FCM) with a compact payload and deep-link to the thread. Push service is rate-limited and coalesces rapid events to avoid spam.

### 9) Encryption & privacy

* **Server-side encryption:** Messages at rest are encrypted on servers using service-side encryption keys; transport uses TLS.
* **End-to-end encryption (E2EE):** Instagram has been rolling out E2E for some chats (opt-in / specific flows) — this means message plaintext is not accessible to server operators in theory. E2E introduces complexities: search in chat, spam detection, backups. Instagram offers secure storage/backup options (secure PIN) for E2E-enabled messages.

> Note: E2E availability and default status have changed over time; platform level docs show Meta actively rolling out E2E features but still keep many features (like some safety scans) working with other designs.

### 10) Safety & moderation in DMs

* **On-device ML:** For sensitive content (nudity, self-harm), Instagram uses on-device ML models to blur or warn teens and to reduce sending harmful media. This allows safety actions even in encrypted chats if the model runs locally.
* **Reporting pipeline:** User reports create events that trigger remote fetch (if allowed) or local evidence upload. Some checks (like scam detection) are server-side and run on metadata/signals.

### 11) Scaling techniques specific to DMs

* **Sharded thread ownership:** Partition threads by `thread_id` hash to owners (set of messaging nodes) to limit hot-shard effects.
* **Coalescing & fanout optimization:** For group chats or viral threads, servers coalesce identical downstream payloads and use multicast-like strategies to reduce duplicate writes.
  -**Backpressure & throttling:** Rate limits per sender/thread prevent abuse during spikes.

### Quick sequence diagram (simplified)

```
Client A --socket--> Edge Gateway --RPC--> Messaging Service -> Durable Store
                                 |--> Push Service (APNs/FCM)
                                 |--> Fanout to recipient gateways
Recipient Gateway --socket--> Client B
```

---

## SEGMENT 9: RELIABILITY + SCALING (95:00 – 105:00)

**Rohit:**
Instagram focuses heavily on reliability.

Techniques:

* Auto‑scaling clusters
* Circuit breakers
* Rate limiting
* Observability dashboards

Metrics tracked:

* p50 / p99 latency
* Error rates
* Cache hit ratios

Failures expected hote hain. Systems designed to degrade gracefully.

---

## SEGMENT 10: KEY LESSONS (105:00 – 115:00)

**Rohit:**
Instagram se kya seekhte hain?

1. **Cache aggressively**
2. **Shard early**
3. **Design for failure**
4. **Measure everything**
5. **Use async pipelines**

Scale comes from architecture, not one magic tool.

---

## CLOSING (115:00 – 120:00)

**Rohit:**
Instagram ek app nahi hai. Ye thousands of distributed systems ka orchestra hai.

Har scroll ke peeche:

* Databases
* Caches
* ML models
* Global networks

Engineering ka goal simple hai:

**"Billions of users. Milliseconds of latency."**

**Aditya:**
Aur tradeoffs?

**Rohit:**
Always. Scalability vs simplicity. Speed vs consistency.

**TOGETHER:**
Pick your tradeoffs wisely.

---

## APPENDIX: QUICK REFERENCE

### Core Components

* CDN + Edge caching
* Microservices architecture
* Redis/Memcached caching
* Sharded PostgreSQL
* Distributed object storage
* ML ranking pipeline
* Event streaming systems

### Key Concepts

* Horizontal scaling
* Cache‑aside pattern
* Sharding via hashing
* Event‑driven architecture
* Feed ranking systems
* Observability metrics

---

**END OF SCRIPT**
