# 🐦 Twitter-Like Feed System Architecture (with Feed Shards + Hybrid Model)

This document outlines the end-to-end architecture of a scalable feed system, such as the one used by Twitter/X, supporting:
- **Sharded feed storage**
- **Pub/Sub messaging**
- **Hybrid push/pull feed distribution**
- **Ranking and ad injection**
- **Multi-region replication**

---

## 📤 When a User Creates a Post

### 🧾 Step-by-Step Flow

#### 1. **User Creates a Post**
- User A sends a new post (tweet/update) via the mobile or web app.
- The app calls the **Post API** via the `Feed Publisher Service`.

#### 2. **Feed Publisher Service**
- Validates the post and metadata.
- Publishes the post into the **First MQ (Feed Topic)** using a system like:
  - Kafka
  - Google Pub/Sub
  - AWS SNS/SQS

---

## 📦 First MQ: `PostEventsTopic`

- **Topic Name:** `post.events.topic`
- **Type:** Pub/Sub
- **Publisher:** `Feed Publisher Service`
- **Consumers:**
  - `Persistent Storage Service`
  - `Regional Forwarder Service`
  - `Ranking Logic Service (S1)`

---

### 🧵 3. Consumers of First MQ

#### 3.1. **Persistent Storage Service**
- Stores the raw post in a **relational DB** (e.g., Postgres, MySQL) or distributed log store.
- This DB serves as the **source of truth** for:
  - Post retrieval
  - Editing/deletion history
  - Backup/restore

#### 3.2. **Regional Forwarder Service**
- Handles cross-region replication.
- Checks if the post needs to be sent to **followers in Region B**.
- Republishes the same post event to **Region B's `post.events.topic`**.

#### 3.3. **S1 — Feed Fan-Out + Ranking Service**
- Fetches all **followers of User A** from the **Social Graph Service**.
- Filters by region — keeps only **same-region followers**.
- Computes **relevance score** per follower (optional).
- Uses consistent hashing (`follower_id % N`) to map each follower to a **Feed Shard**.
- Publishes new post to the correct **Feed Notification Topics (FNTs)**.

---

## 🔔 Second MQ: Feed Notification Topics (FNT)

- **Topic per feed shard** — e.g., `fnt.shard.0`, `fnt.shard.1`, ..., `fnt.shard.N`
- **Publisher:** `S1 Ranking Service`
- **Subscribers:** `Feed Shard Services`

---

## 🧩 Feed Shard Services

Each Feed Shard handles:
- **A subset of users' feeds**, based on hashing of user ID.
- **Subscribes to one FNT** topic (e.g., `fnt.shard.3`).
- **Consumes new post notifications** for its assigned users.
- **Writes new feed entries** into the feed store (can be Redis, Cassandra, DynamoDB).

---

### 🗃 Feed Storage Format

Each user's feed is stored as a **sorted list** of feed items:

```json
{
  "user_id": "1234",
  "feed": [
    {"post_id": "a1", "score": 0.92, "timestamp": "2025-06-18T10:20Z"},
    {"post_id": "b2", "score": 0.81, "timestamp": "2025-06-18T09:30Z"}
  ]
}
```

- Stored in Redis, Cassandra, RocksDB, or another fast key-value store.
- Sorted by computed ranking_score.

## 👀 When a User Loads Their Feed

This section explains how a user's home timeline (feed) is generated on demand, using a **merge-on-read** strategy that combines precomputed content, hot/trending posts, ads, and ranking.

---

### 🧭 Step-by-Step Workflow

#### 1. **User Opens App and Requests Feed**
- The client makes a request to `GET /feed`.
- The request is routed to the **Feed API Gateway** or Load Balancer.

#### 2. **Routing to Feed Generation Cluster**
- Based on the user ID, a consistent hashing function is used to map the request to the correct node in the **Feed Generation Cluster**.
- That node is aware of which **Feed Shard** holds the user's precomputed feed data.

#### 3. **Fetch Feed Components**

The feed generation node pulls together the following inputs:

| Source                      | Description |
|-----------------------------|-------------|
| 🗂 **Feed Shard (User-Specific)** | Pulls precomputed feed entries from fast storage (Redis, Cassandra, DynamoDB, etc.) that were pushed via fan-out from friends' posts. |
| 🔥 **Hot Cache (Celebrity/Viral Content)** | High-fanout posts from celebrity users are not pushed to all followers. Instead, they are placed in a centralized in-memory cache (Redis, Memcached). These are pulled and optionally filtered on read. |
| 💰 **Ad Service** | Ads are injected into the feed. The ad server decides which ads to serve based on targeting, user profile, and campaign goals. |
| 🧠 **Ranking Service** | Merges, filters, and reranks feed entries using ML/heuristic models. Factors may include recency, engagement likelihood, social graph affinity, and ad placement rules. |

#### 4. **Merge and Rerank**

- All retrieved entries are merged into a single feed list.
- The ranking algorithm sorts the merged list using a computed score (`ranking_score`).
- Deduplication and policy filters are applied.

#### 5. **Return Final Feed to Client**

- The final feed list (typically ~20-100 items) is returned to the user.
- Paging or infinite scroll requests can pull more entries, which may include lazy merging of hot content.

---

### 🧠 Ranking Strategy (Simplified)

Ranking typically involves:
- Score = `w1 * freshness + w2 * relevance + w3 * engagement_probability + w4 * virality + ...`
- Could involve ML models trained on:
  - Click-through rate (CTR)
  - Time spent
  - Follow/like/share behavior

---

### 🗃 Temporary Feed Caching

To reduce redundant computation:
- The merged/ranked feed is cached per user (e.g., in Redis) for a short TTL (e.g., 60 seconds).
- Helps handle fast-refresh use cases without recomputing ranking logic.

---

## 🧩 Summary: Merge-on-Read Composition

| Component           | Push or Pull | Notes |
|---------------------|--------------|-------|
| Feed Shard Storage  | Push         | Contains posts from friends, updated via FNT fan-out |
| Hot Cache           | Pull         | Contains viral/celebrity content, fetched on demand |
| Ad Server           | Pull         | Injects relevant sponsored content |
| Ranking Engine      | Pull         | Merges and reranks final feed content per request |

---

## 📈 Benefits of Merge-on-Read Design

- ⚡ Fast reads with precomputed feed entries
- 📉 Avoids write amplification for viral/celebrity content
- 🧠 Dynamically personalized experience using real-time ranking
- 📦 Compact feed storage per user, with efficient sharding

---

## 🧠 Ranking Algorithm Internals

The **ranking engine** is a core component of feed generation. It computes a **relevance score** for each feed entry before final sorting.

### 🧮 Example Scoring Function

```text
ranking_score = 
    (w1 × freshness) +
    (w2 × engagement_probability) +
    (w3 × affinity_with_author) +
    (w4 × content_type_boost) +
    (w5 × virality_score)
```

📊 Factors in the Ranking Model
Factor	Description
Freshness	How recent the post is
Engagement Probability	Based on user's historical behavior (CTR, likes, replies)
Affinity Score	Closeness to the post author (friends, mutuals, DM history)
Content Type Boost	Preference for media (video > image > text), etc.
Virality Score	Likes, shares, or retweets within a short window
Diversity Penalty	Penalizes too many posts from the same user
Ad Priority	Paid content can override organic ranking via weights

🧠 ML Models
Models may be linear (for interpretability) or deep learning-based (for personalization)

Trained on:

Historical click/impression data

Feed session logs

Engagement outcomes (likes, replies, profile views)

🔥 Hot Post Cache (Celebrity/Viral Content)
🔁 Purpose
Avoid pushing posts from high-fanout users (e.g., celebrities) to millions of feeds. Instead, make them available on demand.

🗃 Stored In
In-memory cache: Redis, Memcached, or custom in-memory KV store

Indexed by post_id, timestamp, category, language, or region

⚙️ Lifecycle
New viral post detected (via fan-out threshold or scoring)

Post is inserted into hot cache (with short TTL: 5–15 min)

During feed generation, hot posts are merged in

Ranking engine decides if they’re relevant for the current user

🧠 Benefits
Reduces write amplification

Enables global trending content visibility

Still allows fine-grained ranking per user

💰 Ad Injection Engine
Ads are considered first-class feed entries, injected and ranked alongside organic content.

Ad Fetching Workflow
Feed Generation Service requests ad slots from Ad Server

Ad Server:

Matches campaigns to user

Filters by budget, audience, relevance

Returns ad metadata

Ads are merged with feed entries

Ranking engine assigns scores to both ads and organic items

Final sorting is applied

Ad Constraints
Position caps (e.g., 1 ad every 5 organic posts)

Impression limits (per session or per ad)

Click tracking and analytics handled via instrumentation

🔁 End-to-End Request Flow Summary
text
Copy
Edit
[User Opens App]
   ↓
[Client → GET /feed]
   ↓
[Feed API Gateway]
   ↓
[Feed Generation Cluster Node (via consistent hash)]
   ↓
   ├── Pull from Feed Shard (user’s local feed)
   ├── Pull hot posts from Redis cache
   ├── Request ads from Ad Server
   └── Merge & rerank using Ranking Engine
   ↓
[Final sorted feed → Returned to Client]
🧠 Recap: Why Merge-on-Read + Feed Shards Works
Benefit	Description
✅ Scalable Writes	Only friend-based fan-out is stored in feed shards
✅ Efficient for Celebrities	No need to push to millions of followers
✅ Fresh Feed Experience	Late-stage merge allows hot/trending content and ads
✅ Personalized Ranking	Combines static + dynamic content per session


