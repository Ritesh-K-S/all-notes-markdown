# How Twitter/X Handles the Firehose of Tweets

> **What you'll learn**: How Twitter processes 500+ million tweets per day, builds personalized timelines for 400+ million users in real-time, handles the "thundering herd" during breaking news, and manages fan-out — the defining architectural challenge of social feeds.

---

## Real-Life Analogy

Imagine you're a **personal newspaper editor** for 400 million people. Each person follows different "reporters" (accounts), and your job is to:

- Collect articles from **every reporter** the moment they publish (500M articles/day)
- Build a **custom newspaper** for each of your 400M readers in real-time
- Handle when a famous reporter publishes something — suddenly **100 million readers** all want that article in their newspaper simultaneously
- The newspaper must update **instantly** — no waiting for the morning edition
- Some readers follow 10 reporters, others follow 10,000
- Some reporters have 1 follower, others have 100 million

This is the **fan-out problem** — Twitter's central architectural challenge.

---

## Core Concept Explained Step-by-Step

### Step 1: The Fan-Out Problem — Twitter's Core Challenge

When a user tweets, that tweet needs to appear in the timeline of every follower. This is called **fan-out**:

```
┌─────────────────────────────────────────────────────────────────┐
│              THE FAN-OUT PROBLEM                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Elon Musk tweets "Hello" (180M followers)                       │
│                                                                   │
│  FAN-OUT ON WRITE approach:                                      │
│  Push tweet to all 180M followers' timelines                     │
│  • 180 million write operations!                                 │
│  • Takes too long (seconds to minutes)                           │
│  • Wastes storage (most won't check soon)                        │
│                                                                   │
│  FAN-OUT ON READ approach:                                       │
│  When user opens app, fetch tweets from everyone they follow     │
│  • User follows 500 people? That's 500 queries!                 │
│  • Slow for users who follow many accounts                       │
│  • But no wasted work for inactive users                         │
│                                                                   │
│  TWITTER'S SOLUTION: HYBRID approach                             │
│  • Regular users (< 10K followers): Fan-out on WRITE            │
│  • Celebrities (> 10K followers): Fan-out on READ               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: The Hybrid Timeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│              TWITTER HYBRID TIMELINE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  WHEN A REGULAR USER TWEETS (< 10K followers):                      │
│  ────────────────────────────────────────────                        │
│  Fan-out on Write: Push to each follower's pre-computed timeline    │
│                                                                      │
│  @alice tweets (500 followers)                                       │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────────┐                                               │
│  │  Fan-out Service  │                                               │
│  │  (writes to Redis)│                                               │
│  └──────────┬───────┘                                               │
│             │                                                        │
│      ┌──────┼──────┬──────┬──────┐                                  │
│      ▼      ▼      ▼      ▼      ▼                                  │
│   User1   User2  User3  User4  ...User500                           │
│   Timeline Timeline Timeline Timeline Timeline                       │
│   (Redis)  (Redis) (Redis) (Redis) (Redis)                          │
│                                                                      │
│                                                                      │
│  WHEN A CELEBRITY TWEETS (> 10K followers):                         │
│  ──────────────────────────────────────────                          │
│  NOT fanned out. Mixed in at READ time.                              │
│                                                                      │
│  @elonmusk tweets (180M followers)                                   │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────────┐                                               │
│  │  Stored in tweet  │ (just ONE write)                              │
│  │  storage only     │                                               │
│  └──────────────────┘                                               │
│                                                                      │
│                                                                      │
│  WHEN USER OPENS TIMELINE:                                           │
│  ─────────────────────────                                           │
│                                                                      │
│  ┌──────────────────┐     ┌──────────────────────┐                  │
│  │ Pre-computed      │     │ Celebrity tweets     │                  │
│  │ timeline (Redis)  │  +  │ (fetched on demand)  │                  │
│  │ (regular users'   │     │ (from tweet store)   │                  │
│  │  tweets already   │     │                      │                  │
│  │  pushed here)     │     │                      │                  │
│  └────────┬──────────┘     └───────────┬──────────┘                  │
│           │                            │                             │
│           └────────────┬───────────────┘                             │
│                        ▼                                             │
│              ┌──────────────────┐                                    │
│              │  Merge & Rank    │                                    │
│              │  (by time +      │                                    │
│              │   relevance ML)  │                                    │
│              └──────────────────┘                                    │
│                        │                                             │
│                        ▼                                             │
│              User's personalized timeline                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 3: What Happens When You Tweet

```
User creates tweet: "Just shipped a new feature! 🚀"
         │
         ▼
┌────────────────────────────────────────────────────────────────┐
│ Step 1: WRITE TWEET (synchronous — fast path)                  │
│ • Validate (length, content policy)                            │
│ • Store in tweets table (primary storage)                      │
│ • Generate unique tweet ID (Snowflake ID)                      │
│ • Return success to user immediately                           │
└──────────────────────┬─────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────┐
│ Step 2: ASYNC FAN-OUT (background — may take seconds)          │
│                                                                 │
│ • Fetch follower list                                          │
│ • For each follower (if user has < 10K followers):             │
│   • Append tweet ID to follower's timeline cache (Redis)       │
│   • Keep only last ~800 tweets per timeline                    │
│ • If user has > 10K followers:                                 │
│   • Skip fan-out (will be fetched on read)                     │
│                                                                 │
│ • Push notifications to users with notifications enabled       │
│ • Update search index (tweet now searchable)                   │
│ • Update trending topics aggregation                           │
│ • Process @mentions and #hashtags                              │
└────────────────────────────────────────────────────────────────┘
```

### Step 4: The Timeline Service in Detail

```
┌─────────────────────────────────────────────────────────────────┐
│              TIMELINE SERVICE ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  User opens Twitter app → "Get my timeline"                      │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────┐                                │
│  │  Timeline Mixer Service      │                                │
│  │                              │                                │
│  │  Inputs:                     │                                │
│  │  1. Home timeline cache     ─┼──▶ Redis (pre-computed)        │
│  │  2. Celebrity tweets        ─┼──▶ Tweet store (on demand)     │
│  │  3. Ads                     ─┼──▶ Ad service                  │
│  │  4. Who-to-follow           ─┼──▶ Recommendation service     │
│  │  5. Trending topics         ─┼──▶ Trends service             │
│  └──────────────┬───────────────┘                                │
│                 │                                                  │
│                 ▼                                                  │
│  ┌──────────────────────────────┐                                │
│  │  Ranking Service (ML)        │                                │
│  │                              │                                │
│  │  Signals:                    │                                │
│  │  • Recency (newer = better) │                                │
│  │  • Engagement (likes, RTs)  │                                │
│  │  • Relationship (close      │                                │
│  │    friends score higher)    │                                │
│  │  • Content type preference  │                                │
│  │  • Past interaction history │                                │
│  └──────────────┬───────────────┘                                │
│                 │                                                  │
│                 ▼                                                  │
│  ┌──────────────────────────────┐                                │
│  │  Final Timeline Response     │                                │
│  │  ~20-50 tweets + ads + UI    │                                │
│  └──────────────────────────────┘                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Snowflake — Twitter's Unique ID Generator

Twitter needed globally unique, sortable IDs for tweets. They invented **Snowflake**:

```
┌─────────────────────────────────────────────────────────────────┐
│              SNOWFLAKE ID STRUCTURE (64 bits)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─┬──────────────────────────────┬──────────┬──────────────┐   │
│  │0│    Timestamp (41 bits)       │Worker(10)│Sequence(12)  │   │
│  └─┴──────────────────────────────┴──────────┴──────────────┘   │
│   │              │                      │           │             │
│   │              │                      │           │             │
│   │   Milliseconds since               │    Counter within      │
│   │   Twitter epoch                     │    same millisecond    │
│   │   (gives ~69 years)                 │    (4096 per ms)      │
│   │                                     │                        │
│   │                              Machine/DC ID                   │
│   │                              (1024 workers)                  │
│   Unused                                                         │
│                                                                   │
│  Properties:                                                      │
│  • Time-ordered: Later tweets have bigger IDs                    │
│  • Unique: No coordination needed between servers                │
│  • Fast: Each server generates 4096 IDs per millisecond          │
│  • 64-bit: Fits in a long integer (efficient storage & sort)     │
│                                                                   │
│  Example: 1453868745216098304                                    │
│  You can extract the timestamp FROM the ID itself!               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Twitter's Storage Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TWITTER DATA STORAGE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  TWEET STORE (MySQL → Manhattan)                     │        │
│  │  • All tweets ever created                          │        │
│  │  • Sharded by tweet ID                              │        │
│  │  • Billions of rows                                 │        │
│  │  • Immutable (tweets don't change once created)     │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  TIMELINE CACHE (Redis)                              │        │
│  │  • Pre-computed timelines for each user             │        │
│  │  • Stores last ~800 tweet IDs per user              │        │
│  │  • NOT full tweet content — just IDs                │        │
│  │  • Very fast reads (microseconds)                   │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  SOCIAL GRAPH (FlockDB → custom)                    │        │
│  │  • Who follows whom                                 │        │
│  │  • Billions of edges                                │        │
│  │  • Must answer: "who does @user follow?"            │        │
│  │    and "who follows @user?" quickly                 │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  SEARCH INDEX (Earlybird — custom Lucene)           │        │
│  │  • Real-time search across all tweets               │        │
│  │  • Inverted index, updated within seconds           │        │
│  │  • Handles trending/breaking news search spikes     │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  ENGAGEMENT COUNTERS (Redis + Manhattan)             │        │
│  │  • Likes, retweets, reply counts                    │        │
│  │  • Must handle millions of increments per second    │        │
│  │  • Eventually consistent (it's OK if count is       │        │
│  │    slightly behind for a few seconds)               │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Handling the Thundering Herd (Breaking News)

```
┌─────────────────────────────────────────────────────────────────┐
│       BREAKING NEWS / VIRAL MOMENT HANDLING                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Scenario: World Cup final goal scored                           │
│  Effect: 20M tweets/minute + 200M timeline refreshes            │
│                                                                   │
│  Problem without protection:                                     │
│  • Everyone searches "goal" simultaneously                       │
│  • Everyone refreshes timeline simultaneously                    │
│  • Backend gets 100x normal traffic in seconds                   │
│                                                                   │
│  Twitter's defenses:                                             │
│                                                                   │
│  Layer 1: CDN + Cache (absorb reads)                            │
│  ┌──────────────────────────────────────────────┐               │
│  │  • Trending topics cached aggressively        │               │
│  │  • Same search query cached (TTL: 5-10 sec)  │               │
│  │  • Tweet hydration results cached             │               │
│  └──────────────────────────────────────────────┘               │
│                                                                   │
│  Layer 2: Rate limiting per user                                 │
│  ┌──────────────────────────────────────────────┐               │
│  │  • Can't refresh more than X times/sec       │               │
│  │  • API rate limits enforced at edge           │               │
│  └──────────────────────────────────────────────┘               │
│                                                                   │
│  Layer 3: Backpressure + graceful degradation                   │
│  ┌──────────────────────────────────────────────┐               │
│  │  • Fan-out queue backs up → delays accepted   │               │
│  │  • Non-essential features disabled            │               │
│  │  • "Who to follow" skipped during spikes     │               │
│  │  • Timeline shows slightly stale data         │               │
│  └──────────────────────────────────────────────┘               │
│                                                                   │
│  Layer 4: Auto-scaling                                           │
│  ┌──────────────────────────────────────────────┐               │
│  │  • Pre-scale before known events (elections)  │               │
│  │  • Auto-scale timeline service horizontally   │               │
│  └──────────────────────────────────────────────┘               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### The For You Algorithm (Recommendation)

```
┌─────────────────────────────────────────────────────────────────┐
│         TWITTER "FOR YOU" RANKING PIPELINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Step 1: CANDIDATE GENERATION (1000s of tweets)                  │
│  ┌──────────────────────────────────────────────┐               │
│  │  Sources:                                     │               │
│  │  • In-network: Tweets from people you follow │               │
│  │  • Out-of-network: Tweets liked by people    │               │
│  │    you follow (social proof)                 │               │
│  │  • Topic-based: Tweets matching your         │               │
│  │    interests (inferred from behavior)        │               │
│  │  • Trending: Globally/locally trending       │               │
│  └──────────────────────────────────────────────┘               │
│                          │                                        │
│                          ▼                                        │
│  Step 2: SCORING with ML Model (rank each tweet)                 │
│  ┌──────────────────────────────────────────────┐               │
│  │  Features per tweet:                          │               │
│  │  • Author reputation & your relationship     │               │
│  │  • Content engagement (likes, RTs, replies)  │               │
│  │  • Recency                                    │               │
│  │  • Media type (image/video boost)            │               │
│  │  • Your predicted probability of:            │               │
│  │    - Liking (P_like)                         │               │
│  │    - Retweeting (P_RT)                       │               │
│  │    - Replying (P_reply)                      │               │
│  │    - Spending time reading (P_dwell)         │               │
│  │                                               │               │
│  │  Score = w1*P_like + w2*P_RT + w3*P_reply    │               │
│  │        + w4*P_dwell - penalty_for_negative   │               │
│  └──────────────────────────────────────────────┘               │
│                          │                                        │
│                          ▼                                        │
│  Step 3: FILTERING & DIVERSITY                                   │
│  ┌──────────────────────────────────────────────┐               │
│  │  • Remove content policy violations           │               │
│  │  • Remove muted/blocked content              │               │
│  │  • Ensure diversity (not all from same user) │               │
│  │  • Insert ads at natural positions           │               │
│  │  • Balance in-network vs out-of-network      │               │
│  └──────────────────────────────────────────────┘               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Fan-Out on Write with Redis

```python
# Simplified Twitter fan-out: when a user tweets, push to all followers' timelines
import redis
import time
from typing import List

class TimelineService:
    """
    Demonstrates Twitter's fan-out-on-write pattern.
    Regular user tweets → pushed to all followers' timelines in Redis.
    """
    
    MAX_TIMELINE_SIZE = 800  # Keep last 800 tweets per user
    CELEBRITY_THRESHOLD = 10_000  # Don't fan-out for accounts with 10K+ followers
    
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)
    
    def post_tweet(self, user_id: str, tweet_id: str, follower_ids: List[str]):
        """Handle a new tweet — fan-out to followers."""
        
        # Store the tweet itself (primary storage)
        self.redis.hset(f"tweet:{tweet_id}", mapping={
            "user_id": user_id,
            "tweet_id": tweet_id,
            "timestamp": str(time.time()),
        })
        
        # Decide: fan-out on write or skip?
        if len(follower_ids) > self.CELEBRITY_THRESHOLD:
            # Celebrity: DON'T fan out. Will be fetched on read.
            self.redis.sadd("celebrities", user_id)
            return "stored_only"  # No fan-out for celebrities
        
        # Regular user: Fan-out to each follower's timeline
        for follower_id in follower_ids:
            timeline_key = f"timeline:{follower_id}"
            # Add tweet ID with timestamp as score (sorted set)
            self.redis.zadd(timeline_key, {tweet_id: time.time()})
            # Trim to keep only latest 800
            self.redis.zremrangebyrank(timeline_key, 0, -(self.MAX_TIMELINE_SIZE + 1))
        
        return "fanned_out"
    
    def get_timeline(self, user_id: str, count: int = 50) -> List[str]:
        """
        Get user's timeline: merge pre-computed + celebrity tweets.
        """
        # Get pre-computed timeline (fan-out-on-write results)
        timeline_key = f"timeline:{user_id}"
        tweet_ids = self.redis.zrevrange(timeline_key, 0, count - 1)
        
        # Also fetch latest from celebrities this user follows
        # (fan-out-on-read for celebrity tweets)
        celebrity_tweets = self._fetch_celebrity_tweets(user_id, count=20)
        
        # Merge and sort by timestamp
        all_tweets = list(set(tweet_ids + celebrity_tweets))
        # In production: ML-based ranking here
        return all_tweets[:count]
    
    def _fetch_celebrity_tweets(self, user_id: str, count: int) -> List[str]:
        """Fetch recent tweets from celebrities the user follows."""
        # In production: fetch from tweet store, not cached in timeline
        following = self.redis.smembers(f"following:{user_id}")
        celebrities = self.redis.smembers("celebrities")
        followed_celebs = following & celebrities
        
        tweets = []
        for celeb_id in followed_celebs:
            recent = self.redis.zrevrange(f"user_tweets:{celeb_id}", 0, 5)
            tweets.extend(recent)
        return tweets[:count]
```

### Java — Snowflake ID Generator

```java
/**
 * Twitter's Snowflake ID generator - generates unique, time-sorted 64-bit IDs.
 * No coordination needed between servers.
 * Used for tweet IDs, user IDs, DM IDs, etc.
 */
public class SnowflakeIdGenerator {
    // Twitter epoch: Nov 4, 2010 (custom start time)
    private static final long EPOCH = 1288834974657L;
    
    // Bit allocation
    private static final int WORKER_ID_BITS = 5;
    private static final int DATACENTER_ID_BITS = 5;
    private static final int SEQUENCE_BITS = 12;
    
    // Max values
    private static final long MAX_WORKER_ID = ~(-1L << WORKER_ID_BITS);       // 31
    private static final long MAX_DATACENTER_ID = ~(-1L << DATACENTER_ID_BITS); // 31
    private static final long MAX_SEQUENCE = ~(-1L << SEQUENCE_BITS);           // 4095
    
    // Bit shifts
    private static final int WORKER_SHIFT = SEQUENCE_BITS;
    private static final int DATACENTER_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS;
    private static final int TIMESTAMP_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS + DATACENTER_ID_BITS;
    
    private final long workerId;
    private final long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;

    public SnowflakeIdGenerator(long workerId, long datacenterId) {
        if (workerId > MAX_WORKER_ID || datacenterId > MAX_DATACENTER_ID) {
            throw new IllegalArgumentException("Worker/Datacenter ID out of range");
        }
        this.workerId = workerId;
        this.datacenterId = datacenterId;
    }

    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        
        if (timestamp == lastTimestamp) {
            // Same millisecond: increment sequence
            sequence = (sequence + 1) & MAX_SEQUENCE;
            if (sequence == 0) {
                // Sequence exhausted — wait for next millisecond
                timestamp = waitNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0;  // New millisecond: reset sequence
        }
        
        lastTimestamp = timestamp;
        
        // Compose the 64-bit ID
        return ((timestamp - EPOCH) << TIMESTAMP_SHIFT)
             | (datacenterId << DATACENTER_SHIFT)
             | (workerId << WORKER_SHIFT)
             | sequence;
    }

    private long waitNextMillis(long last) {
        long timestamp = System.currentTimeMillis();
        while (timestamp <= last) {
            timestamp = System.currentTimeMillis();
        }
        return timestamp;
    }

    public static void main(String[] args) {
        SnowflakeIdGenerator gen = new SnowflakeIdGenerator(1, 1);
        long tweetId = gen.nextId();
        System.out.println("Tweet ID: " + tweetId);
        // Output: 1453868745216098304 (unique, time-ordered, 64-bit)
    }
}
```

---

## Infrastructure Examples

### Twitter's Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Timeline Cache** | Redis (massive cluster) | Pre-computed timelines |
| **Tweet Store** | Manhattan (custom K/V store) | Primary tweet storage |
| **Social Graph** | FlockDB → Manhattan | Who follows whom |
| **Search** | Earlybird (custom Lucene) | Real-time tweet search |
| **Messaging** | Kafka + custom (Kestrel → Kafka) | Async event streaming |
| **Compute** | Mesos → Kubernetes | Container orchestration |
| **Cache** | Twemcache (Memcached fork) | General caching |
| **ML** | Custom + TensorFlow | Ranking, recommendations |
| **Monitoring** | Custom (Observability stack) | Metrics, alerting |
| **ID Generation** | Snowflake | Unique ID generation |

### Traffic Flow During Normal Operations

```
┌─────────────────────────────────────────────────────────────────┐
│              TWITTER REQUEST FLOW                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Mobile App / Web Client                                         │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────┐                                            │
│  │  CDN (static)    │  ← Images, JS, CSS                        │
│  └──────────────────┘                                            │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────┐                                            │
│  │  API Gateway     │  ← Auth, rate limiting, routing            │
│  │  (Finagle-based) │                                            │
│  └────────┬─────────┘                                            │
│           │                                                       │
│    ┌──────┴──────┬───────────────┬──────────────┐                │
│    ▼             ▼               ▼              ▼                │
│ ┌────────┐ ┌──────────┐ ┌───────────┐ ┌───────────────┐        │
│ │Timeline│ │  Tweet   │ │  Search   │ │Notifications  │        │
│ │Service │ │  Service │ │  Service  │ │   Service     │        │
│ │        │ │          │ │(Earlybird)│ │               │        │
│ └────────┘ └──────────┘ └───────────┘ └───────────────┘        │
│     │           │             │              │                    │
│     ▼           ▼             ▼              ▼                    │
│ ┌────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐         │
│ │ Redis  │ │Manhattan │ │  Search   │ │  Manhattan   │         │
│ │Timeline│ │(Tweet DB)│ │  Index    │ │  + Redis     │         │
│ │ Cache  │ │          │ │           │ │              │         │
│ └────────┘ └──────────┘ └───────────┘ └──────────────┘         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### The Fail Whale Era (2008-2013) vs. Today

Early Twitter was notorious for the "Fail Whale" error page during high-traffic events. Here's how they fixed it:

| Problem | Old (Fail Whale Era) | New (Post-2013) |
|---------|---------------------|-----------------|
| Architecture | Ruby on Rails monolith | JVM microservices (Scala/Java) |
| Database | MySQL (single) | Manhattan (distributed) |
| Caching | Minimal | Redis + Memcached clusters |
| Fan-out | None (query on read) | Hybrid (write + read) |
| Deployment | Hours of downtime | Rolling deploys |
| Scale | 5K tweets/sec max | 500K+ tweets/sec |

### Twitter During Super Bowl / Elections

- **Super Bowl peak**: 30M+ tweets during the game
- **US Election 2020**: 700K tweets/minute at peak
- Pre-scaling: Twitter adds capacity BEFORE known events
- Feature degradation: "Who to follow" disabled during extreme spikes
- Result: No more Fail Whale (last seen ~2013)

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| Pure fan-out on write for everyone | Celebrity with 100M followers = 100M writes per tweet | Hybrid: fan-out on write for regular, read for celebrities |
| Storing full tweet in timeline cache | Wastes memory; tweet content rarely changes | Store only tweet IDs in timeline; hydrate when serving |
| Auto-incrementing IDs | Can't distribute across servers; leaks information | Snowflake-style distributed ID generation |
| Synchronous fan-out | User waits while 1000s of timeline updates happen | Async: return to user immediately, fan-out in background |
| No timeline size limit | Users who never clean up accumulate unbounded data | Cap at ~800 tweets; older tweets fetched from cold storage |
| Treating all tweets equally in ranking | Users miss important content in chronological-only feeds | ML-based ranking combining engagement signals + relevance |

---

## When to Use / When NOT to Use

### When to Use This Architecture
- Building a social feed with follower-based distribution
- Millions of users creating content consumed by followers
- Real-time or near-real-time delivery requirements
- Highly asymmetric follow graphs (celebrities vs. regular users)
- Need for personalized algorithmic ranking

### When NOT to Use This Architecture
- Private messaging (different pattern — see WhatsApp chapter)
- Small community (< 100K users) → Simple database queries work fine
- Content with long shelf-life (articles) → Different caching strategy
- Symmetrical relationships only (like messaging) → No fan-out needed
- Non-social content (e.g., e-commerce) → Different access patterns

---

## Key Takeaways

1. **The fan-out problem is Twitter's defining challenge**: A tweet from a celebrity (100M followers) vs. a regular user (500 followers) requires completely different strategies
2. **Hybrid approach is the answer**: Fan-out on write for regular users (pre-compute timelines) + fan-out on read for celebrities (fetch on demand). Neither alone works at scale.
3. **Snowflake IDs** solve distributed unique ID generation — time-ordered, unique across servers, no coordination needed, fits in 64 bits
4. **Redis is the backbone** of timeline delivery — each user's timeline is a sorted set of tweet IDs, updated asynchronously
5. **Async processing is essential**: The user gets immediate confirmation; fan-out, search indexing, and notifications happen in the background
6. **Breaking news = thundering herd**: Aggressive caching, rate limiting, and graceful degradation prevent cascading failures during viral moments
7. **ML ranking replaced pure chronological** — predicting which tweets you'll engage with is now core to the product (the "For You" tab)

---

## What's Next?

Next, we'll explore [How Uber/Ola Handles Real-Time Location & Matching](./07-uber-ride-matching.md) — understanding how ride-hailing platforms match riders to drivers in seconds using geospatial indexing, real-time location streams, and dispatch algorithms.
