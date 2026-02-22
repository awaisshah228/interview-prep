# Databases (MongoDB & PostgreSQL) - Interview Q&A

> 20+ questions covering MongoDB aggregation, schema design, PostgreSQL optimization, and general DB concepts

---

## Table of Contents

- [MongoDB](#mongodb)
- [PostgreSQL](#postgresql)
- [General Database Concepts](#general-database-concepts)

---

## MongoDB

### Q1: MongoDB Aggregation Pipeline - Explain with examples.

**Answer:**

```javascript
// Aggregation = series of data processing stages (like Unix pipes)

// Example: Daily trading analytics
db.trades.aggregate([
  // Stage 1: Filter
  { $match: {
    executedAt: { $gte: new Date('2024-01-01') },
    status: 'completed'
  }},

  // Stage 2: Group by trader and day
  { $group: {
    _id: {
      trader: '$traderId',
      day: { $dateToString: { format: '%Y-%m-%d', date: '$executedAt' } }
    },
    dailyPnL: { $sum: '$pnl' },
    tradeCount: { $sum: 1 },
    avgTradeSize: { $avg: '$amount' },
    symbols: { $addToSet: '$symbol' },
    maxTrade: { $max: '$amount' },
  }},

  // Stage 3: Sort
  { $sort: { '_id.day': -1 } },

  // Stage 4: Lookup (JOIN)
  { $lookup: {
    from: 'users',
    localField: '_id.trader',
    foreignField: '_id',
    as: 'traderInfo'
  }},
  { $unwind: '$traderInfo' },

  // Stage 5: Reshape output
  { $project: {
    trader: '$traderInfo.name',
    date: '$_id.day',
    dailyPnL: { $round: ['$dailyPnL', 2] },
    tradeCount: 1,
    avgTradeSize: { $round: ['$avgTradeSize', 2] },
    _id: 0
  }},

  // Stage 6: Limit
  { $limit: 100 }
]);

// Window functions (MongoDB 5.0+)
{ $setWindowFields: {
  partitionBy: '$_id.trader',
  sortBy: { '_id.day': 1 },
  output: {
    movingAvg7d: {
      $avg: '$dailyPnL',
      window: { documents: [-6, 0] }
    },
    runningTotal: {
      $sum: '$dailyPnL',
      window: { documents: ['unbounded', 'current'] }
    },
    rank: { $rank: {} }
  }
}}

// $facet - multiple aggregations in one pass
{ $facet: {
  totalStats: [
    { $group: { _id: null, total: { $sum: '$amount' }, count: { $sum: 1 } } }
  ],
  bySymbol: [
    { $group: { _id: '$symbol', total: { $sum: '$amount' } } },
    { $sort: { total: -1 } }
  ],
  recent: [
    { $sort: { executedAt: -1 } },
    { $limit: 5 }
  ]
}}
```

**Performance tips:**
- Put `$match` and `$project` early (reduces data through pipeline)
- Index fields used in `$match` and `$sort`
- Use `$facet` to avoid multiple queries
- `allowDiskUse: true` for large aggregations (>100MB)
- `$lookup` is expensive — denormalize when possible

---

### Q2: MongoDB Indexing Strategies.

**Answer:**

```javascript
// Types of indexes
// 1. Single field
db.trades.createIndex({ symbol: 1 }); // ascending

// 2. Compound (multi-field) - ORDER MATTERS
db.trades.createIndex({ symbol: 1, executedAt: -1 });
// Supports: { symbol: 1 }, { symbol: 1, executedAt: -1 }
// Does NOT support: { executedAt: -1 } alone

// ESR Rule for compound indexes:
// E = Equality fields first (exact match)
// S = Sort fields next
// R = Range fields last
db.trades.createIndex({ symbol: 1, executedAt: -1, amount: 1 });
//                      Equality      Sort            Range

// 3. Text index (full-text search)
db.articles.createIndex({ title: 'text', body: 'text' });
db.articles.find({ $text: { $search: 'solana trading' } });

// 4. TTL index (auto-delete documents)
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 });

// 5. Partial index (index subset of documents)
db.trades.createIndex(
  { executedAt: -1 },
  { partialFilterExpression: { status: 'completed' } }
);

// 6. Unique index
db.users.createIndex({ email: 1 }, { unique: true });

// 7. Wildcard index (for dynamic fields)
db.events.createIndex({ 'metadata.$**': 1 });

// Analyzing query performance
db.trades.find({ symbol: 'SOL' }).explain('executionStats');
// Look for:
// - IXSCAN (good) vs COLLSCAN (bad - full collection scan)
// - nReturned vs totalDocsExamined (should be close)
// - totalKeysExamined (should be close to nReturned)

// Index intersection: MongoDB can combine multiple single-field indexes
// But compound index is usually more efficient
```

---

### Q3: MongoDB Schema Design - Embedding vs Referencing.

**Answer:**

```javascript
// EMBEDDING (Denormalized) - data stored together
// ✅ Best for: 1:1, 1:few, read-heavy, always accessed together
{
  _id: ObjectId("..."),
  name: "Awais",
  email: "m.awaisshah228@gmail.com",
  // Embedded subdocument (1:1)
  profile: {
    bio: "Senior Software Engineer",
    avatar: "https://...",
  },
  // Embedded array (1:few, bounded)
  addresses: [
    { type: "home", city: "Islamabad" },
    { type: "work", city: "Remote" }
  ]
}

// REFERENCING (Normalized) - separate collections
// ✅ Best for: 1:many, many:many, unbounded, frequently updated independently
// User document
{ _id: ObjectId("user1"), name: "Awais" }
// Trade documents (separate collection)
{ _id: ObjectId("trade1"), userId: ObjectId("user1"), symbol: "SOL", ... }
{ _id: ObjectId("trade2"), userId: ObjectId("user1"), symbol: "ETH", ... }

// HYBRID - embed summary, reference details
{
  _id: ObjectId("user1"),
  name: "Awais",
  // Embedded summary (updated periodically)
  tradeStats: {
    totalTrades: 150,
    totalPnL: 25000,
    winRate: 0.65,
    lastTradeAt: new Date()
  },
  // Full trade history in separate collection
  // Use $lookup or separate query when needed
}

// BUCKET PATTERN - for time-series data
{
  sensorId: "price-sol",
  date: new Date("2024-01-15"),
  measurements: [
    { time: "09:00", value: 95.50 },
    { time: "09:01", value: 95.55 },
    // ... up to 1440 per document (1 per minute)
  ],
  count: 2,
  sum: 191.05,
  min: 95.50,
  max: 95.55
}
```

| Factor | Embed | Reference |
|---|---|---|
| Read together? | Yes → Embed | No → Reference |
| Array grows? | Bounded (<100) → Embed | Unbounded → Reference |
| Updated often? | Rarely → Embed | Often → Reference |
| Doc size? | < 16MB → Embed | Large → Reference |
| Need atomic? | Yes → Embed | Multi-doc txn → Reference |

---

### Q4: MongoDB Transactions.

**Answer:**

```javascript
// MongoDB supports multi-document ACID transactions (4.0+ replica set, 4.2+ sharded)

const session = client.startSession();

try {
  session.startTransaction({
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' },
  });

  // All operations in the transaction
  const trade = await trades.insertOne(
    { symbol: 'SOL', amount: 100, userId: 'user1', price: 95 },
    { session }
  );

  await balances.updateOne(
    { userId: 'user1' },
    { $inc: { available: -9500, locked: 9500 } },
    { session }
  );

  await orderBook.deleteOne(
    { orderId: 'order1' },
    { session }
  );

  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  session.endSession();
}

// Mongoose transaction
const session = await mongoose.startSession();
await session.withTransaction(async () => {
  await Trade.create([tradeData], { session });
  await Balance.updateOne(filter, update, { session });
});

// When to use transactions:
// ✅ Financial operations (trade execution, balance updates)
// ✅ Multi-document consistency required
// ❌ Single document updates (already atomic)
// ❌ High-throughput writes (transactions add latency)

// Performance impact:
// - Extra round trips to replica set
// - Locks held for duration
// - Default 60s timeout
// - Keep transactions SHORT
```

---

### Q5: MongoDB Change Streams.

**Answer:**

```javascript
// Change Streams = real-time notifications for data changes
// Requires replica set or sharded cluster

// Watch all changes on a collection
const changeStream = db.collection('trades').watch([
  { $match: { operationType: { $in: ['insert', 'update'] } } },
  { $match: { 'fullDocument.symbol': 'SOL' } }
], {
  fullDocument: 'updateLookup', // include full document on update
});

changeStream.on('change', (change) => {
  console.log('Change type:', change.operationType);
  console.log('Document:', change.fullDocument);
  console.log('Update:', change.updateDescription); // for updates

  // Broadcast to WebSocket clients
  wss.broadcast(JSON.stringify({
    type: 'trade_update',
    data: change.fullDocument,
  }));
});

// Resume after disconnect (using resume token)
const resumeToken = change._id; // save this
const newStream = collection.watch([], { resumeAfter: resumeToken });

// Use cases:
// 1. Real-time dashboards (trade updates → WebSocket → UI)
// 2. Cache invalidation (MongoDB change → clear Redis cache)
// 3. Sync to search engine (MongoDB → Elasticsearch)
// 4. Event sourcing / audit logging
// 5. Cross-service data sync

// NestJS integration
@Injectable()
export class TradeChangeStreamService implements OnModuleInit {
  async onModuleInit() {
    const collection = this.connection.collection('trades');
    const stream = collection.watch();

    stream.on('change', (change) => {
      this.eventEmitter.emit('trade.changed', change);
    });
  }
}
```

---

## PostgreSQL

### Q6: PostgreSQL Query Optimization with EXPLAIN ANALYZE.

**Answer:**

```sql
-- EXPLAIN shows the query plan
-- EXPLAIN ANALYZE actually runs the query and shows real timing

EXPLAIN ANALYZE
SELECT t.*, u.name
FROM trades t
JOIN users u ON t.user_id = u.id
WHERE t.symbol = 'SOL'
  AND t.executed_at > '2024-01-01'
ORDER BY t.executed_at DESC
LIMIT 50;

-- Reading the output:
-- Seq Scan          → Full table scan (BAD for large tables)
-- Index Scan        → Using index (GOOD)
-- Index Only Scan   → Even better (covers all columns)
-- Bitmap Index Scan → Combining multiple index results
-- Nested Loop       → Good for small result sets
-- Hash Join         → Good for large result sets
-- Sort              → Expensive if not backed by index

-- Key metrics:
-- actual time=0.5..12.3  → startup..total time in ms
-- rows=50                → actual rows returned
-- loops=1                → number of executions
-- Planning Time: 0.2ms
-- Execution Time: 12.5ms

-- OPTIMIZATION STRATEGIES:

-- 1. Create proper indexes
CREATE INDEX idx_trades_symbol_date
ON trades (symbol, executed_at DESC)
INCLUDE (user_id, amount, price); -- covering index (Index Only Scan)

-- 2. Partial index (smaller, faster)
CREATE INDEX idx_active_trades ON trades (executed_at DESC)
WHERE status = 'active';

-- 3. Expression index
CREATE INDEX idx_trades_lower_symbol ON trades (LOWER(symbol));
-- Now: WHERE LOWER(symbol) = 'sol' uses index

-- 4. ANALYZE - update table statistics for better plans
ANALYZE trades;

-- 5. Avoid SELECT * (fetch only needed columns)
SELECT t.id, t.symbol, t.amount FROM trades t; -- not SELECT *

-- 6. Use LIMIT with ORDER BY (stop after N rows)

-- 7. CTEs can be optimization barriers (before PG 12)
-- PG 12+: CTEs are inlined by default unless WITH ... AS MATERIALIZED
```

---

### Q7: PostgreSQL Window Functions.

**Answer:**

```sql
-- 1. Running total per user
SELECT
  user_id, executed_at, amount,
  SUM(amount) OVER (
    PARTITION BY user_id
    ORDER BY executed_at
    ROWS UNBOUNDED PRECEDING
  ) AS running_total
FROM trades;

-- 2. Ranking traders
SELECT
  user_id, total_pnl,
  RANK() OVER (ORDER BY total_pnl DESC) AS rank,          -- gaps after ties
  DENSE_RANK() OVER (ORDER BY total_pnl DESC) AS dense_rank, -- no gaps
  ROW_NUMBER() OVER (ORDER BY total_pnl DESC) AS row_num,    -- unique
  NTILE(4) OVER (ORDER BY total_pnl DESC) AS quartile        -- divide into N groups
FROM trader_stats;

-- 3. Moving average
SELECT
  date, daily_pnl,
  AVG(daily_pnl) OVER (
    ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS moving_avg_7d,
  AVG(daily_pnl) OVER (
    ORDER BY date
    ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
  ) AS moving_avg_30d
FROM daily_stats;

-- 4. Lead/Lag - compare with previous/next
SELECT
  symbol, executed_at, price,
  LAG(price, 1) OVER w AS prev_price,
  LEAD(price, 1) OVER w AS next_price,
  price - LAG(price, 1) OVER w AS price_change,
  ROUND(
    (price - LAG(price, 1) OVER w) / LAG(price, 1) OVER w * 100, 2
  ) AS pct_change
FROM trades
WINDOW w AS (PARTITION BY symbol ORDER BY executed_at);

-- 5. OHLC (Open-High-Low-Close) candlestick data
SELECT DISTINCT
  symbol,
  DATE_TRUNC('hour', executed_at) AS period,
  FIRST_VALUE(price) OVER w AS open,
  MAX(price) OVER w AS high,
  MIN(price) OVER w AS low,
  LAST_VALUE(price) OVER w AS close,
  SUM(amount) OVER w AS volume
FROM trades
WINDOW w AS (
  PARTITION BY symbol, DATE_TRUNC('hour', executed_at)
  ORDER BY executed_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);

-- 6. Percent of total
SELECT
  symbol,
  SUM(amount) AS total_amount,
  ROUND(
    SUM(amount) * 100.0 / SUM(SUM(amount)) OVER (), 2
  ) AS pct_of_total
FROM trades
GROUP BY symbol
ORDER BY total_amount DESC;
```

---

### Q8: PostgreSQL Transactions and Isolation Levels.

**Answer:**

```sql
-- Transaction isolation levels (from least to most strict):

-- 1. READ UNCOMMITTED (PostgreSQL treats as READ COMMITTED)
--    Can see uncommitted changes from other transactions

-- 2. READ COMMITTED (default in PostgreSQL)
--    Only sees committed data, but may see different data on re-read
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 3. REPEATABLE READ
--    Snapshot at start of transaction, same data on re-read
--    Prevents: dirty reads, non-repeatable reads
--    Doesn't prevent: phantom reads (in theory, PG prevents them too)
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 4. SERIALIZABLE
--    Transactions behave as if executed sequentially
--    Prevents everything, but may throw serialization errors (retry needed)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

```
Phenomena:
─────────────────────────────────────────────────
Dirty Read:       Read uncommitted data (rolled back later)
Non-repeatable:   Same query returns different data (another txn committed)
Phantom Read:     Same query returns different ROWS (insert by another txn)

Level           | Dirty | Non-repeatable | Phantom
────────────────┼───────┼────────────────┼────────
READ COMMITTED  | No    | Yes            | Yes
REPEATABLE READ | No    | No             | No (in PG)
SERIALIZABLE    | No    | No             | No
```

```sql
-- Practical: Trade execution with proper isolation
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Check balance
SELECT available_balance FROM wallets WHERE user_id = 'user1' FOR UPDATE;
-- FOR UPDATE locks the row (prevents concurrent modification)

-- Execute trade
INSERT INTO trades (user_id, symbol, amount, price) VALUES ('user1', 'SOL', 10, 95);

-- Update balance
UPDATE wallets SET available_balance = available_balance - 950 WHERE user_id = 'user1';

COMMIT;
```

---

### Q9: PostgreSQL Partitioning.

**Answer:**

```sql
-- Partitioning = split large table into smaller physical tables
-- Query planner automatically routes to correct partition

-- 1. RANGE partitioning (most common for time-series)
CREATE TABLE trades (
  id SERIAL,
  symbol VARCHAR(10),
  amount DECIMAL(18,8),
  price DECIMAL(18,8),
  executed_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (id, executed_at)
) PARTITION BY RANGE (executed_at);

-- Create partitions
CREATE TABLE trades_2024_q1 PARTITION OF trades
  FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE trades_2024_q2 PARTITION OF trades
  FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
CREATE TABLE trades_2024_q3 PARTITION OF trades
  FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');
CREATE TABLE trades_2024_q4 PARTITION OF trades
  FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

-- Default partition (catches rows that don't match any range)
CREATE TABLE trades_default PARTITION OF trades DEFAULT;

-- 2. LIST partitioning
CREATE TABLE trades_by_exchange (
  id SERIAL,
  exchange VARCHAR(20),
  ...
) PARTITION BY LIST (exchange);

CREATE TABLE trades_binance PARTITION OF trades_by_exchange FOR VALUES IN ('binance');
CREATE TABLE trades_coinbase PARTITION OF trades_by_exchange FOR VALUES IN ('coinbase');

-- 3. HASH partitioning (even distribution)
CREATE TABLE trades_hash (
  id SERIAL,
  user_id INTEGER,
  ...
) PARTITION BY HASH (user_id);

CREATE TABLE trades_h0 PARTITION OF trades_hash FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE trades_h1 PARTITION OF trades_hash FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ...

-- Benefits:
-- ✅ Partition pruning (only scan relevant partitions)
-- ✅ Faster DELETE (drop partition instead of row-by-row)
-- ✅ Parallel query execution across partitions
-- ✅ Easier maintenance (VACUUM individual partitions)

-- Auto-create partitions (pg_partman extension)
SELECT partman.create_parent(
  p_parent_table := 'public.trades',
  p_control := 'executed_at',
  p_type := 'native',
  p_interval := '1 month'
);
```

---

### Q10: PostgreSQL JSON/JSONB Operations.

**Answer:**

```sql
-- JSONB = binary JSON (faster queries, supports indexing)
-- JSON = text JSON (preserves formatting, no indexing)

CREATE TABLE events (
  id SERIAL PRIMARY KEY,
  type VARCHAR(50),
  metadata JSONB NOT NULL DEFAULT '{}'
);

-- Insert
INSERT INTO events (type, metadata) VALUES
('trade', '{"symbol": "SOL", "price": 95.50, "tags": ["defi", "spot"]}');

-- Query
SELECT * FROM events
WHERE metadata->>'symbol' = 'SOL';              -- text extraction
WHERE metadata->'price' > '90';                   -- JSON comparison
WHERE metadata @> '{"symbol": "SOL"}';            -- contains
WHERE metadata ? 'tags';                           -- has key
WHERE metadata ?| array['symbol', 'amount'];       -- has any key
WHERE metadata ?& array['symbol', 'price'];        -- has all keys
WHERE metadata->'tags' @> '"defi"';               -- array contains

-- Nested access
SELECT metadata->'nested'->'deep'->>'value' FROM events;
-- or
SELECT metadata #>> '{nested,deep,value}' FROM events;

-- Update
UPDATE events SET metadata = metadata || '{"status": "completed"}'; -- merge
UPDATE events SET metadata = metadata - 'temporary_field';           -- remove key
UPDATE events SET metadata = jsonb_set(metadata, '{price}', '100'); -- set specific path

-- Indexing JSONB
CREATE INDEX idx_events_metadata ON events USING GIN (metadata);           -- all keys/values
CREATE INDEX idx_events_symbol ON events ((metadata->>'symbol'));           -- specific field
CREATE INDEX idx_events_path ON events USING GIN (metadata jsonb_path_ops); -- @> operator only

-- Aggregation
SELECT
  metadata->>'symbol' AS symbol,
  COUNT(*) AS trade_count,
  AVG((metadata->>'price')::numeric) AS avg_price
FROM events
WHERE type = 'trade'
GROUP BY metadata->>'symbol';
```

---

## General Database Concepts

### Q11: ACID Properties.

**Answer:**

```
A - Atomicity:    All or nothing. Transaction fully completes or fully rolls back.
C - Consistency:  Database moves from one valid state to another.
I - Isolation:    Concurrent transactions don't interfere with each other.
D - Durability:   Committed data survives crashes (written to disk).

Example (trade execution):
BEGIN TRANSACTION;
  1. Deduct balance from buyer     ← If step 3 fails,
  2. Add tokens to buyer           ← all steps are
  3. Deduct tokens from seller     ← rolled back
  4. Add balance to seller         ← (Atomicity)
COMMIT;

PostgreSQL: Full ACID support
MongoDB: ACID for single-document ops, multi-document transactions since 4.0
Redis: Not ACID (but has transactions with MULTI/EXEC for atomicity)
```

---

### Q12: CAP Theorem.

**Answer:**

```
CAP = you can only guarantee 2 of 3 in a distributed system:

C - Consistency:    Every read gets the most recent write
A - Availability:   Every request gets a response (may not be latest)
P - Partition tolerance: System works despite network failures

In practice, P is non-negotiable (networks WILL fail), so you choose:

CP (Consistency + Partition tolerance):
  → When network splits, some nodes reject requests to stay consistent
  → Examples: PostgreSQL (single primary), MongoDB, HBase, Redis (single)
  → Good for: Financial data, trade execution

AP (Availability + Partition tolerance):
  → When network splits, all nodes respond but may have stale data
  → Examples: Cassandra, DynamoDB, CouchDB
  → Good for: Social media feeds, analytics, shopping carts

Real world: It's a spectrum, not binary
  → DynamoDB: AP by default, but can request strong consistency per query
  → MongoDB: CP with read preference options
```

---

### Q13: N+1 Query Problem.

**Answer:**

```typescript
// PROBLEM: Fetching related data in a loop
async function getTradesWithUsers() {
  const trades = await Trade.find(); // 1 query
  for (const trade of trades) {
    trade.user = await User.findById(trade.userId); // N queries!
  }
  // Total: 1 + N queries (if 100 trades → 101 queries!)
}

// SOLUTION 1: Eager loading / JOIN (SQL)
SELECT t.*, u.name FROM trades t
JOIN users u ON t.user_id = u.id;
// 1 query total

// SOLUTION 2: Populate (Mongoose)
const trades = await Trade.find().populate('user'); // 2 queries (1 + 1 IN query)

// SOLUTION 3: DataLoader (batch + cache)
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (userIds) => {
  const users = await User.find({ _id: { $in: userIds } });
  const userMap = new Map(users.map(u => [u.id.toString(), u]));
  return userIds.map(id => userMap.get(id.toString()));
});

// Now: userLoader.load(trade.userId) batches all calls into 1 query
// Also caches within the same request

// SOLUTION 4: TypeORM relations
const trades = await tradeRepo.find({
  relations: ['user'],
  // or with QueryBuilder:
  // .leftJoinAndSelect('trade.user', 'user')
});

// SOLUTION 5: Prisma includes
const trades = await prisma.trade.findMany({
  include: { user: true },
});
```

---

### Q14: Database Normalization vs Denormalization.

**Answer:**

```
NORMALIZATION (reduce redundancy):
──────────────────────────────────
1NF: No repeating groups, atomic values
2NF: 1NF + no partial dependencies (all non-key columns depend on full key)
3NF: 2NF + no transitive dependencies (non-key depends only on key)

users:          trades:
id | name       id | user_id | symbol | amount
1  | Awais      1  | 1       | SOL    | 100

✅ No data duplication
✅ Easy updates (change in one place)
✅ Data integrity
❌ Complex queries (many JOINs)
❌ Slower reads

DENORMALIZATION (add redundancy for speed):
──────────────────────────────────────────
trades:
id | user_id | user_name | symbol | amount | symbol_price
1  | 1       | Awais     | SOL    | 100    | 95.50

✅ Faster reads (no JOINs)
✅ Simpler queries
❌ Data duplication
❌ Update anomalies (change name → update everywhere)
❌ More storage

WHEN TO DENORMALIZE:
- Read-heavy workloads (dashboards, analytics)
- Performance is critical
- Data rarely changes
- MongoDB (denormalization is natural)

WHEN TO NORMALIZE:
- Write-heavy workloads
- Data integrity is critical
- Storage is a concern
- PostgreSQL (normalization is natural)
```

---

### Q15: Connection Pooling - Why and how?

**Answer:**

```typescript
// Problem: Creating a new DB connection per request is expensive
// ~50-100ms to establish TCP + TLS + auth per connection
// Under load → thousands of connections → DB crashes

// Solution: Connection Pool (reuse connections)

// PostgreSQL with node-postgres
import { Pool } from 'pg';
const pool = new Pool({
  host: 'localhost',
  database: 'trades',
  user: 'app',
  password: 'secret',
  max: 20,               // maximum connections in pool
  min: 5,                // minimum idle connections
  idleTimeoutMillis: 30000,  // close idle connections after 30s
  connectionTimeoutMillis: 2000, // fail if can't connect in 2s
});

// Usage: pool automatically manages connections
const result = await pool.query('SELECT * FROM trades');
// Connection is returned to pool after query

// TypeORM pool config
TypeOrmModule.forRoot({
  type: 'postgres',
  extra: {
    max: 20,
    min: 5,
  },
});

// External pooler: PgBouncer
// Sits between app and PostgreSQL
// Modes:
// - Session pooling: connection per session (default)
// - Transaction pooling: connection per transaction (most efficient)
// - Statement pooling: connection per statement

// MongoDB connection pooling (built into driver)
mongoose.connect(uri, {
  maxPoolSize: 100,     // max connections per server
  minPoolSize: 10,      // min connections
  maxIdleTimeMS: 30000, // close idle after 30s
});

// Sizing rule of thumb:
// pool_size = (CPU cores * 2) + effective_spindle_count
// For SSD: ~10-20 connections for most apps
// More connections != better performance (contention increases)
```

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [Node.js & NestJS](./03-nodejs-nestjs.md) | **Next**: [System Design](./05-system-design.md)
