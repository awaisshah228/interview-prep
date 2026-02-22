# System Design & Architecture - Interview Q&A

> 15+ questions covering design patterns, scalability, real system design problems

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Design Patterns](#design-patterns)
- [System Design Problems](#system-design-problems)

---

## Core Concepts

### Q1: How do you approach a system design interview?

**Answer:**

```
FRAMEWORK (45 minutes):

1. REQUIREMENTS (5 min)
   ├─ Functional: What does the system DO?
   ├─ Non-functional: Scale, latency, availability, consistency?
   ├─ Constraints: Budget, team size, timeline?
   └─ Back-of-envelope: Users, QPS, storage, bandwidth

2. HIGH-LEVEL DESIGN (10 min)
   ├─ Draw major components
   ├─ Define APIs (REST/gRPC/GraphQL)
   ├─ Data flow between components
   └─ Choose database type

3. DETAILED DESIGN (20 min)
   ├─ Database schema
   ├─ Deep dive into 2-3 critical components
   ├─ Handle edge cases
   └─ Scaling strategies

4. TRADE-OFFS & BOTTLENECKS (10 min)
   ├─ Identify bottlenecks
   ├─ Discuss alternatives
   ├─ Monitoring & observability
   └─ Future improvements
```

**Back-of-envelope calculations:**
```
Key numbers to remember:
- 1 day = ~86,400 seconds ≈ ~100K seconds
- 1 million requests/day ≈ 12 requests/second
- QPS to plan for = average QPS × 2-3 (peak factor)
- Read:Write ratio: Most apps are 10:1 or higher
- 1 char = 1 byte, 1 KB = 1000 chars, 1 MB = 1000 KB
- 1 million users × 1 KB each = 1 GB
```

---

### Q2: Explain Microservices vs Monolith.

**Answer:**

```
MONOLITH                              MICROSERVICES
────────────                          ──────────────
┌─────────────────┐                   ┌────────┐  ┌────────┐  ┌────────┐
│    Monolith     │                   │ Auth   │  │ Trade  │  │ Notif  │
│  ┌───────────┐  │                   │Service │  │Service │  │Service │
│  │   Auth    │  │                   └───┬────┘  └───┬────┘  └───┬────┘
│  ├───────────┤  │                       │           │           │
│  │  Trades   │  │        vs         ┌───┴───────────┴───────────┴────┐
│  ├───────────┤  │                   │         Message Bus / API GW   │
│  │  Notifs   │  │                   └────────────────────────────────┘
│  └───────────┘  │
│   Single DB     │                   Each service has its own DB
└─────────────────┘
```

| Factor | Monolith | Microservices |
|---|---|---|
| Team size | < 10 devs | > 10, multiple teams |
| Deploy | Single unit, simple | Independent, complex |
| Scaling | Entire app | Per service |
| Data | Single DB (ACID easy) | Distributed (eventual consistency) |
| Debug | Easy (single process) | Hard (distributed tracing) |
| Dev speed | Fast initially | Slower setup, faster at scale |
| Failure | Entire app down | Partial failure possible |

**Rule:** Start monolith, extract microservices when needed (Monolith First).

---

### Q3: Explain the SNS + SQS Fan-out Pattern.

**Answer:**

```
                         ┌─────────────────┐
  Trade Executed ───────▶│   SNS Topic     │
                         │ "trade-events"  │
                         └────────┬────────┘
                    Filter:       │         Filter:
                    symbol=SOL    │         all events
                    ┌─────────────┼─────────────┐──────────┐
                    ▼             ▼              ▼          ▼
             ┌───────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐
             │ SQS Queue │ │ SQS Queue │ │SQS Queue │ │SQS Queue │
             │ Analytics │ │  Notify   │ │  Audit   │ │Risk Check│
             │    DLQ ──▶│ │    DLQ ──▶│ │   DLQ ──▶│ │   DLQ ──▶│
             └─────┬─────┘ └─────┬─────┘ └────┬─────┘ └────┬─────┘
                   ▼             ▼             ▼            ▼
             ┌───────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐
             │ Analytics │ │ Notif.    │ │ Audit    │ │ Risk     │
             │ Lambda    │ │ Service   │ │ Service  │ │ Service  │
             └───────────┘ └───────────┘ └──────────┘ └──────────┘
```

```typescript
// Publisher
async publishTradeEvent(trade: Trade) {
  await this.sns.publish({
    TopicArn: TRADE_TOPIC_ARN,
    Message: JSON.stringify(trade),
    MessageAttributes: {
      eventType: { DataType: 'String', StringValue: 'TRADE_EXECUTED' },
      symbol: { DataType: 'String', StringValue: trade.symbol },
    },
  });
}

// SNS Subscription Filter (only SOL trades to specific queue)
{
  "symbol": ["SOL"],
  "eventType": ["TRADE_EXECUTED"]
}
```

**Benefits:**
- **Decoupling** — publisher doesn't know about consumers
- **Independent scaling** — each queue scales independently
- **Fault tolerance** — DLQ catches failures, auto-retry
- **Filtering** — reduce unnecessary processing
- **Backpressure** — SQS buffers messages during spikes

---

### Q4: Explain Caching Strategies.

**Answer:**

```
CACHING PATTERNS:

1. CACHE-ASIDE (Lazy Loading) — Most common
   ┌──────┐     miss     ┌──────┐     ┌──────┐
   │ App  │──────────────▶│ Cache│     │  DB  │
   │      │◀──────────────│(miss)│     │      │
   │      │───────────────────────────▶│      │
   │      │◀───────────────────────────│      │
   │      │──set──────────▶│ Cache│     │      │
   └──────┘               └──────┘     └──────┘

   Code:
   async getUser(id) {
     let user = await redis.get(`user:${id}`);
     if (!user) {
       user = await db.findUser(id);
       await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
     }
     return user;
   }

2. WRITE-THROUGH — Write to cache AND DB simultaneously
   Pro: Cache always up-to-date
   Con: Higher write latency

3. WRITE-BEHIND (Write-Back) — Write to cache, async write to DB
   Pro: Fastest writes
   Con: Data loss risk if cache crashes

4. READ-THROUGH — Cache handles DB reads automatically
   Pro: Application logic simpler
   Con: Initial cache miss latency

CACHE INVALIDATION (hardest problem in CS):
- TTL (Time-To-Live): Simple, eventual consistency
- Event-based: Invalidate on write (SNS → cache clear)
- Version-based: Cache key includes version number

WHAT TO CACHE:
✅ Expensive queries (aggregations, JOINs)
✅ Frequently accessed data (user sessions, config)
✅ External API responses (price feeds, exchange rates)
✅ Computed results (PnL calculations, rankings)
❌ Highly dynamic data (real-time prices → use pub/sub instead)
❌ Large datasets (use CDN for static assets)
```

---

### Q5: Explain Load Balancing Strategies.

**Answer:**

```
                    ┌──────────────┐
     Clients ──────▶│ Load Balancer│
                    └──────┬───────┘
                    ┌──────┼──────┐
                    ▼      ▼      ▼
               ┌──────┐┌──────┐┌──────┐
               │ App1 ││ App2 ││ App3 │
               └──────┘└──────┘└──────┘

ALGORITHMS:
1. Round Robin      — Equal distribution, simple
2. Weighted RR      — Bigger servers get more traffic
3. Least Connections — Send to least busy server
4. IP Hash          — Same client → same server (sticky sessions)
5. Random           — Simple, surprisingly effective

AWS Load Balancers:
- ALB (Application): Layer 7, HTTP/HTTPS, path-based routing
  Use for: Web apps, microservices, WebSocket
- NLB (Network): Layer 4, TCP/UDP, ultra-low latency
  Use for: Gaming, IoT, extreme performance
- CLB (Classic): Legacy, avoid for new apps

Health Checks:
  - Active: LB pings /health endpoint periodically
  - Passive: LB monitors response errors
  - Unhealthy threshold: Remove after N failures
  - Healthy threshold: Re-add after N successes
```

---

## Design Patterns

### Q6: Circuit Breaker Pattern.

**Answer:**

```
States:
         success
  ┌─────────────────┐
  ▼                 │
CLOSED ──failure──▶ OPEN ──timeout──▶ HALF-OPEN
  ▲     (threshold)  │                   │
  │                   │    success        │
  │                   │◀──────────────────┘
  │                   │    failure
  │                   └──────▶ OPEN
  │
  └──── All working normally
```

```typescript
class CircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private failureCount = 0;
  private lastFailureTime: number = 0;

  constructor(
    private readonly threshold: number = 5,
    private readonly timeout: number = 30000,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}

// Usage
const priceApiBreaker = new CircuitBreaker(5, 30000);

async function getPrice(symbol: string) {
  return priceApiBreaker.execute(() =>
    fetch(`https://api.prices.com/${symbol}`).then(r => r.json())
  );
}
// After 5 failures → circuit opens → returns error immediately for 30s
// After 30s → tries one request (half-open)
// If success → closes circuit, normal operation resumes
```

**Use cases:** External API calls, database connections, microservice communication.

---

### Q7: Saga Pattern for Distributed Transactions.

**Answer:**

```
Problem: In microservices, you can't use a single DB transaction
Solution: Saga = sequence of local transactions with compensating actions

CHOREOGRAPHY (event-driven):
────────────────────────────
Trade Service → "TradeCreated" event
    ↓
Balance Service → deducts balance → "BalanceDeducted" event
    ↓
Token Service → transfers tokens → "TokensTransferred" event
    ↓
Trade Service → marks trade as completed

If Token Service fails → "TokenTransferFailed" event
    ↓
Balance Service → compensating action: refund balance
    ↓
Trade Service → marks trade as failed

ORCHESTRATION (central coordinator):
────────────────────────────────────
┌──────────────────┐
│  Trade Saga      │
│  Orchestrator    │
├──────────────────┤
│ 1. Create trade  │──→ Trade Service
│ 2. Deduct balance│──→ Balance Service
│ 3. Transfer token│──→ Token Service
│ 4. Complete trade│──→ Trade Service
│                  │
│ On failure:      │
│ Compensate steps │
│ in reverse order │
└──────────────────┘
```

```typescript
// Orchestration example
class TradeSaga {
  private steps: SagaStep[] = [];

  addStep(execute: () => Promise<void>, compensate: () => Promise<void>) {
    this.steps.push({ execute, compensate });
  }

  async run() {
    const executedSteps: SagaStep[] = [];

    try {
      for (const step of this.steps) {
        await step.execute();
        executedSteps.push(step);
      }
    } catch (error) {
      // Compensate in reverse order
      for (const step of executedSteps.reverse()) {
        await step.compensate().catch(console.error);
      }
      throw error;
    }
  }
}

// Usage
const saga = new TradeSaga();
saga.addStep(
  () => tradeService.create(trade),
  () => tradeService.cancel(trade.id),
);
saga.addStep(
  () => balanceService.deduct(userId, amount),
  () => balanceService.refund(userId, amount),
);
saga.addStep(
  () => tokenService.transfer(from, to, amount),
  () => tokenService.transfer(to, from, amount), // reverse transfer
);
await saga.run();
```

**Choreography vs Orchestration:**
| Factor | Choreography | Orchestration |
|---|---|---|
| Coupling | Loose (events) | Central coordinator |
| Complexity | Hard to track flow | Clear sequence |
| Debugging | Distributed logs | Centralized state |
| Best for | Simple, few steps | Complex, many steps |

---

## System Design Problems

### Q8: Design a Real-time Trading Platform.

**Answer:**

```
Requirements:
- Place buy/sell orders (market, limit, stop)
- Real-time price updates
- Order book management
- Trade history and analytics
- 10K concurrent users, 1K trades/second peak

Architecture:
════════════════════════════════════════════════════════

┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│   Next.js    │────▶│  CloudFront  │────▶│   ALB            │
│   Client     │◀────│     CDN      │     └────────┬─────────┘
└──────────────┘     └──────────────┘              │
      │ WSS                               ┌───────┴────────┐
      ▼                                   ▼                ▼
┌──────────────┐                   ┌────────────┐  ┌────────────┐
│  WebSocket   │                   │ API Service│  │ API Service│
│  Server      │                   │ (NestJS)   │  │ (NestJS)   │
│  (Pusher/    │                   └──────┬─────┘  └──────┬─────┘
│   custom)    │                          │               │
└──────────────┘                   ┌──────┴───────────────┘
                                   │
                            ┌──────┴──────┐
                            │  Order      │
                            │  Matching   │──▶ Redis (Order Book)
                            │  Engine     │
                            └──────┬──────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
             ┌──────────┐  ┌──────────┐  ┌──────────────┐
             │PostgreSQL │  │ SNS Topic│  │    Redis     │
             │(Trades,   │  │(Fan-out) │  │(Cache,Price) │
             │ Orders)   │  └────┬─────┘  └──────────────┘
             └──────────┘       │
                         ┌──────┼──────┐
                         ▼      ▼      ▼
                      SQS:   SQS:   SQS:
                    Notify  Audit  Analytics

Database Schema:
─────────────────
orders:     id, user_id, symbol, type, side, price, amount, status, created_at
trades:     id, buy_order_id, sell_order_id, symbol, price, amount, executed_at
balances:   user_id, currency, available, locked
price_ticks: symbol, price, volume, timestamp (time-series / partitioned)

Key Design Decisions:
1. PostgreSQL for orders/trades (ACID for financial data)
2. Redis for order book (fast reads, sorted sets for price levels)
3. WebSocket for real-time prices (sub-100ms latency)
4. SNS + SQS for post-trade processing (your proven pattern)
5. ECS Fargate for auto-scaling API services

Scaling:
- Horizontal: API servers behind ALB
- Read replicas: PostgreSQL for analytics queries
- Redis Cluster: High-throughput price caching
- SQS: Backpressure during high-volume periods
- Partitioned tables: Trades by month for historical queries
```

---

### Q9: Design a Notification System.

**Answer:**

```
Requirements:
- Multi-channel: Email, Push (mobile), SMS, In-app, WebSocket
- User preferences (opt-in/opt-out per channel)
- Templates with variables
- Rate limiting per user
- Delivery tracking and retry
- Priority levels

Architecture:
════════════════════════════════════════════════════════

┌──────────────┐     ┌──────────────┐
│ Services     │────▶│  SNS Topic   │
│ (trade,auth, │     │ "notify"     │
│  system)     │     └──────┬───────┘
└──────────────┘            │ Filter by priority
                     ┌──────┼──────┐
                     ▼      ▼      ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Critical │ │   High   │ │  Normal  │
              │ SQS      │ │ SQS      │ │ SQS      │
              └────┬─────┘ └────┬─────┘ └────┬─────┘
                   │            │            │
                   └────────────┼────────────┘
                                ▼
                   ┌────────────────────────┐
                   │ Notification           │
                   │ Orchestrator (NestJS)  │
                   │ - Check user prefs     │
                   │ - Apply rate limits    │
                   │ - Render templates     │
                   │ - Route to channels    │
                   └───────────┬────────────┘
                   ┌───────────┼────────────┐──────────┐
                   ▼           ▼            ▼          ▼
           ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
           │  Email   │ │  Push    │ │   SMS    │ │  In-App  │
           │ (SES)    │ │(FCM/APNS)│ │(Twilio)  │ │(WebSocket│
           └──────────┘ └──────────┘ └──────────┘ │+ DB)     │
                                                   └──────────┘

Data Model:
notifications:
  id, user_id, type, channel, template_id, payload, status,
  priority, sent_at, delivered_at, read_at, retry_count

notification_preferences:
  user_id, channel, enabled, quiet_hours_start, quiet_hours_end

notification_templates:
  id, name, subject_template, body_template, channel

Rate Limit Rules:
  - Max 5 emails/hour per user
  - Max 20 push/hour per user
  - Critical notifications bypass rate limits
  - Quiet hours: batch non-critical, send in morning
```

---

### Q10: Design a URL Shortener.

**Answer:**

```
Requirements:
- Shorten long URLs (100M URLs/month)
- Redirect with low latency (<50ms)
- Custom aliases optional
- Click analytics
- URL expiration

Scale:
- 100M new URLs/month = ~40 URLs/second
- Read:Write = 100:1 → 4000 reads/second
- 5 years storage = 6B URLs × 500 bytes = ~3TB

Architecture:
════════════════════════════════════════
┌────────┐    ┌──────┐    ┌──────────────┐
│ Client │───▶│ CDN  │───▶│ API (NestJS) │
└────────┘    └──────┘    └──────┬───────┘
                                 │
                    ┌────────────┼───────────┐
                    ▼            ▼           ▼
             ┌──────────┐ ┌──────────┐ ┌──────────┐
             │  Redis   │ │PostgreSQL│ │  Kafka   │
             │ (cache)  │ │ (source  │ │(analytics│
             │          │ │ of truth)│ │ events)  │
             └──────────┘ └──────────┘ └──────────┘

Short URL generation:
1. Base62 encoding of auto-increment ID
   ID: 12345 → Base62: "dnh" → url: short.ly/dnh

2. Or: Hash (MD5/SHA) + take first 7 chars
   Collision handling: check DB, append counter if exists

3. Or: Pre-generate random IDs in batches (avoid hotspot)

Encoding: Base62 (a-z, A-Z, 0-9)
7 chars = 62^7 = 3.5 trillion possible URLs

Redirect flow:
1. GET short.ly/abc123
2. Check Redis cache → if hit, redirect 301/302
3. Cache miss → query PostgreSQL → cache result → redirect
4. Async: publish click event to Kafka for analytics

301 vs 302:
- 301 (Permanent): Browser caches, fewer server hits, lose analytics
- 302 (Temporary): Always hits server, accurate analytics ✅
```

---

### Q11: Design a Chat Application.

**Answer:**

```
Requirements:
- 1:1 and group messaging
- Real-time delivery
- Message history
- Online presence
- Read receipts
- File sharing
- 100K concurrent users

Architecture:
════════════════════════════════════════

┌────────────┐     ┌──────────────────┐
│   Client   │◀───▶│   WebSocket      │
│ (React)    │     │   Gateway        │
└────────────┘     │   (NestJS)       │
                   └────────┬─────────┘
                            │
               ┌────────────┼────────────┐
               ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Redis   │ │ Message  │ │ Presence │
        │  Pub/Sub │ │ Service  │ │ Service  │
        │ (route   │ │          │ │ (Redis)  │
        │  msgs)   │ └────┬─────┘ └──────────┘
        └──────────┘      │
                    ┌─────┴──────┐
                    ▼            ▼
             ┌──────────┐ ┌──────────┐
             │ Cassandra│ │   S3     │
             │ (messages)│ │ (files)  │
             └──────────┘ └──────────┘

Message flow:
1. User A sends message via WebSocket
2. Gateway → Message Service → store in Cassandra
3. Look up User B's WebSocket server (Redis)
4. Redis Pub/Sub → User B's Gateway → WebSocket → User B
5. If User B offline → push notification via SNS

Data Model (Cassandra):
messages_by_conversation:
  conversation_id (partition key)
  message_id (clustering key, TimeUUID for ordering)
  sender_id, content, type, created_at

conversations_by_user:
  user_id (partition key)
  last_message_at (clustering key, DESC)
  conversation_id, last_message_preview

Why Cassandra:
- Write-heavy (many messages)
- Time-series data (messages ordered by time)
- Horizontal scaling
- High availability (AP in CAP)
```

---

## References & Deep Dive Resources

### System Design Fundamentals
| Topic | Resource |
|---|---|
| System Design Primer | [donnemartin/system-design-primer](https://github.com/donnemartin/system-design-primer) — Best free resource |
| Designing Data-Intensive Applications | [DDIA Book (Martin Kleppmann)](https://dataintensive.net/) — The bible of system design |
| Alex Xu - System Design | [bytebytego.com](https://bytebytego.com/) — Best paid course |
| ByteByteGo YouTube | [ByteByteGo (YouTube)](https://www.youtube.com/@ByteByteGo) — Free system design videos |
| Grokking System Design | [educative.io - Grokking](https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers) |

### Architecture Patterns
| Topic | Resource |
|---|---|
| Microservices | [microservices.io](https://microservices.io/) — Patterns catalog by Chris Richardson |
| Event-Driven Architecture | [AWS - Event-Driven Architecture](https://aws.amazon.com/event-driven-architecture/) |
| CQRS | [Martin Fowler - CQRS](https://martinfowler.com/bliki/CQRS.html) |
| Event Sourcing | [Martin Fowler - Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) |
| Saga Pattern | [microservices.io - Saga](https://microservices.io/patterns/data/saga.html) |
| Circuit Breaker | [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html) |
| API Gateway | [microservices.io - API Gateway](https://microservices.io/patterns/apigateway.html) |
| DDD (Domain-Driven Design) | [DDD Reference by Eric Evans](https://www.domainlanguage.com/ddd/reference/) |

### Scalability
| Topic | Resource |
|---|---|
| Load Balancing | [NGINX - Load Balancing](https://www.nginx.com/resources/glossary/load-balancing/) |
| Caching Strategies | [AWS - Caching Best Practices](https://aws.amazon.com/caching/best-practices/) |
| Database Sharding | [Vitess - Sharding](https://vitess.io/docs/concepts/shard/) |
| CDN | [Cloudflare - What is a CDN?](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/) |
| Message Queues | [AWS - Message Queuing](https://aws.amazon.com/message-queue/) |

### System Design Interview Problems
| Topic | Resource |
|---|---|
| Design YouTube | [ByteByteGo - YouTube](https://www.youtube.com/watch?v=jPKTo1iGQiE) |
| Design WhatsApp | [Gaurav Sen - WhatsApp (YouTube)](https://www.youtube.com/watch?v=vvhC64hQZMk) |
| Design Twitter | [System Design Primer - Twitter](https://github.com/donnemartin/system-design-primer/tree/master#design-the-twitter-timeline-and-search) |
| Design URL Shortener | [System Design Primer - URL Shortener](https://github.com/donnemartin/system-design-primer/tree/master#design-pastebin) |
| High Scalability Blog | [highscalability.com](http://highscalability.com/) — Real-world architecture case studies |

### Diagramming Tools
| Topic | Resource |
|---|---|
| Excalidraw | [excalidraw.com](https://excalidraw.com/) — Whiteboard for system design |
| draw.io | [draw.io](https://app.diagrams.net/) — Free diagramming |
| Mermaid | [mermaid.js.org](https://mermaid.js.org/) — Markdown-based diagrams |

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [Databases](./04-databases.md) | **Next**: [AWS Cloud](./06-aws.md)
