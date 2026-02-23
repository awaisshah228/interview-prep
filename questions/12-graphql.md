# GraphQL - Interview Q&A

> 22 questions covering schema design, resolvers, performance, subscriptions, and advanced patterns

---

## Table of Contents

- [GraphQL Fundamentals](#graphql-fundamentals)
- [Schema Design](#schema-design)
- [Resolvers & Data Fetching](#resolvers--data-fetching)
- [Performance & Optimization](#performance--optimization)
- [Authentication & Authorization](#authentication--authorization)
- [Subscriptions & Real-Time](#subscriptions--real-time)
- [Advanced Patterns](#advanced-patterns)
- [GraphQL vs REST](#graphql-vs-rest)

---

## GraphQL Fundamentals

### Q1: What is GraphQL and how does it differ from REST?

**Answer:**

GraphQL is a query language for APIs and a runtime for executing those queries, developed by Facebook in 2012 and open-sourced in 2015.

| Feature | REST | GraphQL |
|---|---|---|
| Endpoints | Multiple (`/users`, `/posts`) | Single (`/graphql`) |
| Data fetching | Fixed response structure | Client specifies exact fields |
| Over-fetching | Common | Never |
| Under-fetching | Common (multiple round-trips) | Never (nested queries) |
| Versioning | URL-based (`/v1`, `/v2`) | Schema evolution (deprecate fields) |
| Caching | HTTP caching (easy) | Requires extra setup (APQ) |
| Error handling | HTTP status codes | Always 200, errors in response body |
| Type system | Optional (OpenAPI) | Built-in (SDL) |
| Real-time | Separate (WebSocket/SSE) | Built-in (Subscriptions) |

```graphql
# REST: 3 separate requests
# GET /users/1
# GET /users/1/posts
# GET /users/1/followers

# GraphQL: 1 request
query {
  user(id: "1") {
    name
    email
    posts { title createdAt }
    followers { name }
  }
}
```

---

### Q2: Explain the three root operation types.

**Answer:**

```graphql
# 1. Query — READ (like GET), runs in parallel
query GetUser {
  user(id: "1") { name email }
}

# 2. Mutation — WRITE (like POST/PUT/DELETE), runs sequentially
mutation CreateUser {
  createUser(input: { name: "Awais", email: "awais@example.com" }) {
    id
    name
  }
}

# 3. Subscription — REAL-TIME over WebSocket
subscription OnNewTrade {
  tradeCreated(symbol: "SOL") {
    id
    price
    amount
    timestamp
  }
}
```

**Key difference:** Multiple mutations in one request run **sequentially** (to avoid race conditions), while multiple queries run **in parallel**.

---

### Q3: Explain Variables, Fragments, and Directives.

**Answer:**

```graphql
# Variables — parameterize queries safely (never string-interpolate!)
query GetUser($id: ID!, $includeEmail: Boolean = false) {
  user(id: $id) {
    name
    email @include(if: $includeEmail)
  }
}
# JSON: { "id": "1", "includeEmail": true }

# Named Fragments — reusable field selections
fragment UserFields on User {
  id
  name
  email
  avatar
}

query {
  currentUser { ...UserFields role }
  user(id: "2") { ...UserFields posts { title } }
}

# Inline Fragments — for union/interface types
query SearchResults {
  search(term: "GraphQL") {
    __typename
    ... on User { name email }
    ... on Post { title body }
    ... on Comment { body author { name } }
  }
}

# Built-in directives
query($showEmail: Boolean!, $hideAge: Boolean!) {
  user(id: "1") {
    name
    email @include(if: $showEmail)   # include field if true
    age   @skip(if: $hideAge)        # skip field if true
  }
}
```

---

### Q4: What are GraphQL scalar types and how do you create custom ones?

**Answer:**

```graphql
# Built-in scalars
type User {
  id:       ID!       # unique identifier (serialized as String)
  name:     String!
  age:      Int!      # 32-bit signed integer
  balance:  Float!    # double-precision float
  isActive: Boolean!
}

# Custom scalars declared in schema
scalar DateTime
scalar JSON
scalar Email
```

```typescript
// Implementing a custom scalar
import { GraphQLScalarType, Kind } from 'graphql';

const DateTimeScalar = new GraphQLScalarType({
  name: 'DateTime',
  description: 'ISO 8601 date-time string',

  serialize(value: Date): string {        // Server → Client
    return value.toISOString();
  },
  parseValue(value: string): Date {       // Client variable → Server
    return new Date(value);
  },
  parseLiteral(ast): Date {               // Inline literal → Server
    if (ast.kind === Kind.STRING) return new Date(ast.value);
    throw new Error('DateTime must be a string');
  },
});

// NestJS code-first
@Scalar('DateTime')
export class DateTimeScalar implements CustomScalar<string, Date> {
  parseValue(value: string) { return new Date(value); }
  serialize(value: Date)    { return value.toISOString(); }
  parseLiteral(ast: ValueNode) {
    if (ast.kind === Kind.STRING) return new Date(ast.value);
    return null;
  }
}
```

---

## Schema Design

### Q5: Explain Interfaces, Unions, and Enums.

**Answer:**

```graphql
# Enum — fixed set of values
enum TradeStatus { PENDING  EXECUTED  CANCELLED  FAILED }
enum Role        { USER  ADMIN  MODERATOR }

# Interface — shared fields all implementing types must have
interface Node {
  id:        ID!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type User implements Node {
  id: ID!; createdAt: DateTime!; updatedAt: DateTime!
  name: String!; email: String!
}

type Post implements Node {
  id: ID!; createdAt: DateTime!; updatedAt: DateTime!
  title: String!; author: User!
}

# Union — one of several types, NO shared fields required
union SearchResult = User | Post | Comment

type Query {
  node(id: ID!): Node              # generic by interface
  search(term: String!): [SearchResult!]!
}
```

```graphql
# Querying a union — must use inline fragments
query {
  search(term: "Awais") {
    __typename
    ... on User { name email }
    ... on Post { title body }
  }
}
```

| | Interface | Union |
|---|---|---|
| Shared fields | Required | None |
| Use case | Types with common behaviour | Completely different types |
| Example | `Node { id }` | `SearchResult = User \| Post` |

---

### Q6: What is the Input type and why does it exist separately?

**Answer:**

```graphql
# Input types — only used for arguments (mutations/queries)
input CreateUserInput {
  name:  String!
  email: String!
  role:  Role = USER      # default value allowed
}

input UpdateUserInput {
  name:  String
  email: String
}

input PaginationInput {
  page:      Int = 1
  limit:     Int = 20
  sortBy:    String = "createdAt"
  sortOrder: SortOrder = DESC
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
}
```

**Why separate?**
- Regular `type` can have resolvers, circular references, and server-generated fields (`id`, timestamps)
- `input` must be plain serializable data — no resolvers, no interfaces
- Keeps a clear boundary between what goes **in** vs what comes **out**

---

### Q7: Explain Relay-style (Connection) pagination.

**Answer:**

```graphql
type Query {
  users(
    first:  Int     # forward: first N items
    after:  String  # cursor: items after this
    last:   Int     # backward: last N items
    before: String  # cursor: items before this
  ): UserConnection!
}

type UserConnection {
  edges:      [UserEdge!]!
  pageInfo:   PageInfo!
  totalCount: Int!
}

type UserEdge {
  node:   User!    # the actual data
  cursor: String!  # opaque cursor for this item
}

type PageInfo {
  hasNextPage:     Boolean!
  hasPreviousPage: Boolean!
  startCursor:     String
  endCursor:       String
}
```

```typescript
// Server implementation
async function resolveUsers({ first = 20, after }) {
  const afterId = after ? Buffer.from(after, 'base64').toString() : null;
  const where   = afterId ? { id: { $gt: afterId } } : {};
  const rows    = await User.find(where).limit(first + 1).sort({ id: 1 });
  const hasNext = rows.length > first;
  const edges   = rows.slice(0, first).map(u => ({
    node:   u,
    cursor: Buffer.from(u.id).toString('base64'),
  }));
  return {
    edges,
    pageInfo: { hasNextPage: hasNext, endCursor: edges.at(-1)?.cursor },
    totalCount: await User.countDocuments(),
  };
}
```

| | Cursor (Relay) | Offset |
|---|---|---|
| Consistency | Stable on inserts/deletes | Breaks on inserts |
| Performance | O(1) with index | O(n) for large offsets |
| Random page access | Not possible | Possible |
| Best for | Infinite scroll, real-time feeds | Traditional page numbers |

---

## Resolvers & Data Fetching

### Q8: Explain the resolver function signature and execution flow.

**Answer:**

```typescript
// Signature: (parent, args, context, info)
const resolvers = {
  Query: {
    user: (parent, args, context, info) => {
      // parent:  result of parent resolver (null for root Query)
      // args:    query arguments e.g. { id: "1" }
      // context: shared per-request object (auth, DB, loaders)
      // info:    query AST (field selection, path, etc.)
      return context.db.users.findById(args.id);
    },
  },

  User: {
    posts: (parent, args, context) =>
      context.db.posts.findByUserId(parent.id),  // parent = User object

    fullName: (parent) =>
      `${parent.firstName} ${parent.lastName}`,  // computed field

    // If no resolver defined, GraphQL returns parent[fieldName] automatically
  },
};
```

**Execution is top-down, depth-first:**
```
Query.user          →  returns User
  User.name         →  parent = User, returns string
  User.posts        →  parent = User, returns [Post]
    Post.title      →  parent = Post, returns string
    Post.comments   →  parent = Post, returns [Comment]
      Comment.body  →  parent = Comment, returns string
```

---

### Q9: What is the N+1 problem and how does DataLoader solve it?

**Answer:**

```graphql
# This query causes N+1 DB queries
query {
  users {         # 1 query: SELECT * FROM users (returns 100 rows)
    name
    posts {       # 100 queries: SELECT * FROM posts WHERE userId = ?
      title       # 1 per user = 101 total!
    }
  }
}
```

```typescript
import DataLoader from 'dataloader';

// DataLoader batches all .load() calls within one event-loop tick
const postsByUserLoader = new DataLoader(async (userIds: string[]) => {
  // Single query for ALL users at once
  const posts = await Post.find({ userId: { $in: userIds } });

  // Group by userId — MUST return in same order as input keys
  const map = new Map<string, Post[]>();
  posts.forEach(p => {
    const list = map.get(p.userId) || [];
    list.push(p);
    map.set(p.userId, list);
  });
  return userIds.map(id => map.get(id) || []);
});

// Resolver uses loader
const resolvers = {
  User: {
    posts: (parent, _, context) =>
      context.loaders.postsByUser.load(parent.id),
    // All .load() calls in one tick are batched → single DB query
  },
};

// Create fresh loaders PER REQUEST (important: prevents cross-user cache)
function createContext({ req }) {
  return {
    loaders: { postsByUser: new DataLoader(batchPostsByUser) },
    user: authenticate(req),
  };
}
// Result: 100 users → 2 queries instead of 101
```

---

### Q10: Schema-first vs Code-first — when to use each?

**Answer:**

**Schema-first (SDL files):**
```graphql
# user.graphql
type User {
  id:    ID!
  name:  String!
  posts: [Post!]!
}
type Query {
  user(id: ID!): User
}
```
```typescript
// resolvers.ts — implemented separately
const resolvers = {
  Query: { user: (_, { id }) => db.users.findById(id) },
};
```

**Code-first (NestJS / TypeGraphQL):**
```typescript
@ObjectType()
class User {
  @Field(() => ID)  id: string;
  @Field()          name: string;
}

@Resolver(() => User)
class UserResolver {
  @Query(() => User, { nullable: true })
  async user(@Args('id', { type: () => ID }) id: string) {
    return this.userService.findById(id);
  }
}
// Schema is AUTO-GENERATED from decorators
```

| | Schema-first | Code-first |
|---|---|---|
| Source of truth | `.graphql` SDL files | TypeScript decorators |
| Type safety | Needs codegen (graphql-codegen) | Built-in |
| Frontend/design collaboration | Easy (edit SDL directly) | Requires code knowledge |
| Schema/resolver drift | Possible | Impossible |
| Best for | API-first, multi-team, design-first | TypeScript backends, NestJS |

---

## Performance & Optimization

### Q11: How do you prevent abusive or deeply nested queries?

**Answer:**

```typescript
// 1. Depth limiting
import depthLimit from 'graphql-depth-limit';
const server = new ApolloServer({
  validationRules: [depthLimit(5)],
});

// 2. Query complexity analysis
import { createComplexityRule, simpleEstimator } from 'graphql-query-complexity';
const server = new ApolloServer({
  validationRules: [
    createComplexityRule({
      maximumComplexity: 1000,
      estimators: [simpleEstimator({ defaultComplexity: 1 })],
      onComplete: (complexity) => console.log('Complexity:', complexity),
    }),
  ],
});

// 3. Persisted queries — only allow known queries in production
//    Client sends SHA256 hash; server looks up the query by hash.
//    Unknown hashes are rejected → no arbitrary queries allowed.

// 4. Resolver timeout
const resolvers = {
  Query: {
    heavyReport: (_, args) =>
      Promise.race([
        runReport(args),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('Timeout')), 5000)),
      ]),
  },
};
```

---

### Q12: Explain caching strategies in GraphQL.

**Answer:**

```typescript
// 1. Response caching (Apollo Server plugin)
import responseCachePlugin from '@apollo/server-plugin-response-cache';
const server = new ApolloServer({ plugins: [responseCachePlugin()] });

// Schema hints
type Query {
  publicPosts: [Post!]! @cacheControl(maxAge: 300)
}
type Post @cacheControl(maxAge: 60) {
  viewCount: Int! @cacheControl(maxAge: 0)   # never cache
}

// 2. DataLoader — per-request in-memory cache (auto, see Q9)

// 3. Redis application cache
const resolvers = {
  Query: {
    user: async (_, { id }, ctx) => {
      const key = `user:${id}`;
      const cached = await ctx.redis.get(key);
      if (cached) return JSON.parse(cached);
      const user = await ctx.db.users.findById(id);
      await ctx.redis.setex(key, 300, JSON.stringify(user));
      return user;
    },
  },
};

// 4. Apollo Client normalized cache (client-side)
const client = new ApolloClient({
  cache: new InMemoryCache({
    typePolicies: {
      User: { keyFields: ['id'] },   // normalize by id
    },
  }),
});
```

| Strategy | Scope | Best for |
|---|---|---|
| HTTP/CDN | Network | Public read-heavy queries (via APQ + GET) |
| Response cache | Server | Identical repeated queries |
| DataLoader | Per-request | N+1 prevention |
| Redis | Application | DB query results |
| Apollo Client | Client | UI state normalization |

---

## Authentication & Authorization

### Q13: How do you implement auth in GraphQL?

**Answer:**

```typescript
// 1. Authentication — context factory
function createContext({ req }): Context {
  const token = req.headers.authorization?.replace('Bearer ', '');
  let user = null;
  if (token) {
    try { user = jwt.verify(token, process.env.JWT_SECRET); } catch {}
  }
  return { user, loaders: createLoaders() };
}

// 2a. Authorization — in resolver
const resolvers = {
  Query: {
    adminData: (_, __, ctx) => {
      if (!ctx.user)                 throw new AuthenticationError('Not authenticated');
      if (ctx.user.role !== 'ADMIN') throw new ForbiddenError('Not authorized');
      return getAdminData();
    },
  },
};

// 2b. Authorization — NestJS guard (cleaner)
@Resolver(() => User)
export class UserResolver {
  @UseGuards(GqlAuthGuard)
  @Query(() => User)
  me(@CurrentUser() user: User) { return user; }

  @UseGuards(GqlAuthGuard, RolesGuard)
  @Roles('ADMIN')
  @Query(() => [User])
  allUsers() { return this.userService.findAll(); }
}

// 2c. Field-level authorization
const resolvers = {
  User: {
    email: (parent, _, ctx) =>
      ctx.user?.id === parent.id || ctx.user?.role === 'ADMIN'
        ? parent.email
        : null,
  },
};
```

---

## Subscriptions & Real-Time

### Q14: How do GraphQL Subscriptions work?

**Answer:**

```typescript
// Schema
// type Subscription { tradeCreated(symbol: String): Trade! }

import { PubSub, withFilter } from 'graphql-subscriptions';
const pubsub = new PubSub(); // use RedisPubSub in production

const resolvers = {
  Subscription: {
    tradeCreated: {
      subscribe: withFilter(
        (_, { symbol }) => pubsub.asyncIterableIterator(`TRADE_${symbol || 'ALL'}`),
        (payload, variables) =>
          !variables.symbol || payload.tradeCreated.symbol === variables.symbol,
      ),
    },
  },

  Mutation: {
    createTrade: async (_, { input }, ctx) => {
      const trade = await ctx.db.trades.create(input);
      pubsub.publish(`TRADE_${trade.symbol}`, { tradeCreated: trade });
      pubsub.publish('TRADE_ALL',             { tradeCreated: trade });
      return trade;
    },
  },
};

// Production: Redis PubSub for multi-instance support
import { RedisPubSub } from 'graphql-redis-subscriptions';
const pubsub = new RedisPubSub({
  publisher:  new Redis(REDIS_URL),
  subscriber: new Redis(REDIS_URL),
});
```

```typescript
// Client (Apollo Client React hook)
const { data } = useSubscription(gql`
  subscription OnTrade($symbol: String!) {
    tradeCreated(symbol: $symbol) { id price amount timestamp }
  }
`, { variables: { symbol: 'SOL' } });
```

| Protocol | Library | Notes |
|---|---|---|
| WebSocket (modern) | `graphql-ws` | Recommended |
| WebSocket (legacy) | `subscriptions-transport-ws` | Deprecated, avoid |
| SSE | `graphql-sse` | Simpler, HTTP-based |

---

## Advanced Patterns

### Q15: Explain GraphQL Federation.

**Answer:**

Federation splits a GraphQL API across multiple services while presenting a **unified schema** to clients.

```graphql
# Users service (subgraph)
type User @key(fields: "id") {
  id: ID!; name: String!; email: String!
}
type Query { user(id: ID!): User }

# Posts service (subgraph) — extends User from Users service
type Post @key(fields: "id") {
  id: ID!; title: String!; author: User!
}
extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post!]!          # add field to User owned by another service
}

# Orders service
extend type User @key(fields: "id") {
  id: ID! @external
  orders: [Order!]!
}
```

```typescript
// Users service — resolve references from other services
const resolvers = {
  User: {
    __resolveReference(ref) { return usersDB.findById(ref.id); },
  },
};

// Posts service
const resolvers = {
  Post: {
    author: (post) => ({ __typename: 'User', id: post.authorId }),
  },
  User: {
    posts: (user) => postsDB.findByAuthorId(user.id),
  },
};
```

**Key directives:**
- `@key` — entity's primary key for cross-service references
- `@external` — field defined in another service
- `@requires` — resolver needs fields fetched from parent service
- `__resolveReference` — how a service loads an entity by key

---

### Q16: How do you handle errors in GraphQL?

**Answer:**

```typescript
// GraphQL error format (HTTP 200, errors in body)
// { "data": { "user": null }, "errors": [{ "message": "...", "extensions": { "code": "NOT_FOUND" } }] }

import { GraphQLError } from 'graphql';

class NotFoundError extends GraphQLError {
  constructor(resource: string, id: string) {
    super(`${resource} ${id} not found`, {
      extensions: { code: 'NOT_FOUND', statusCode: 404 },
    });
  }
}

class ValidationError extends GraphQLError {
  constructor(message: string, field?: string) {
    super(message, {
      extensions: { code: 'VALIDATION_ERROR', statusCode: 400, field },
    });
  }
}

// Usage
const resolvers = {
  Query: {
    user: async (_, { id }, ctx) => {
      const user = await ctx.db.users.findById(id);
      if (!user) throw new NotFoundError('User', id);
      return user;
    },
  },
};

// Hide internal errors in production
const server = new ApolloServer({
  formatError: (formatted, original) => {
    if (formatted.extensions?.code === 'INTERNAL_SERVER_ERROR') {
      console.error(original);
      return { message: 'Something went wrong', extensions: { code: 'INTERNAL_ERROR' } };
    }
    return formatted;
  },
});
```

---

### Q17: GraphQL with NestJS — code-first setup.

**Answer:**

```typescript
// app.module.ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  subscriptions: { 'graphql-ws': true },
  context: ({ req }) => ({ req }),
})

// user.model.ts
@ObjectType()
export class User {
  @Field(() => ID)  id: string;
  @Field()          name: string;
  @Field()          email: string;
  @Field(() => Date) createdAt: Date;
}

// create-user.input.ts
@InputType()
export class CreateUserInput {
  @Field() @IsNotEmpty() @MinLength(2) name: string;
  @Field() @IsEmail()                  email: string;
}

// user.resolver.ts
@Resolver(() => User)
export class UserResolver {
  constructor(
    private userService: UserService,
    private postService: PostService,
  ) {}

  @Query(() => User, { nullable: true })
  user(@Args('id', { type: () => ID }) id: string) {
    return this.userService.findById(id);
  }

  @Mutation(() => User)
  @UseGuards(GqlAuthGuard)
  createUser(@Args('input') input: CreateUserInput) {
    return this.userService.create(input);
  }

  @ResolveField(() => [Post])   // field resolver — called per User
  posts(@Parent() user: User, @Context() ctx: GqlContext) {
    return ctx.loaders.postsByUser.load(user.id);
  }

  @Subscription(() => User)
  userCreated() {
    return this.pubSub.asyncIterableIterator('USER_CREATED');
  }
}
```

---

### Q18: What are Persisted Queries and why use them?

**Answer:**

```
Normal:    POST /graphql  { "query": "query GetUser($id:ID!){user(id:$id){name email posts{title}}}", "variables": {...} }
APQ:       POST /graphql  { "extensions": { "persistedQuery": { "sha256Hash": "abc123" } }, "variables": {...} }
           If miss → PersistedQueryNotFound
           Retry:  POST /graphql  { "query": "...", "extensions": { "persistedQuery": { "sha256Hash": "abc123" } } }
```

```typescript
// Apollo Server
const server = new ApolloServer({
  persistedQueries: {
    cache: redisCache,
    ttl: 900,
  },
});

// Apollo Client
const link = createPersistedQueryLink({ sha256 }).concat(httpLink);
```

**Benefits:**
- Smaller payloads (hash vs full query string)
- Security — allowlist only known query hashes in production
- CDN caching — deterministic GET requests become cacheable
- Bandwidth savings on mobile

---

### Q19: How do you handle file uploads?

**Answer:**

```typescript
// Option A: graphql-upload (file through GraphQL server)
const resolvers = {
  Upload: GraphQLUpload,
  Mutation: {
    uploadFile: async (_, { file }) => {
      const { createReadStream, filename, mimetype } = await file;
      const url = await uploadToS3(createReadStream(), filename, mimetype);
      return { url, filename, mimetype };
    },
  },
};

// Option B: Pre-signed S3 URL (recommended for large files)
// 1. Client calls mutation → gets S3 pre-signed PUT URL
// 2. Client uploads directly to S3 (bypasses GraphQL server)
// 3. Client calls confirmUpload mutation with the final URL

const resolvers = {
  Mutation: {
    getUploadUrl: async (_, { filename, contentType }) => {
      const key = `uploads/${uuid()}-${filename}`;
      const uploadUrl = await s3.getSignedUrlPromise('putObject', {
        Bucket: process.env.S3_BUCKET,
        Key: key, ContentType: contentType, Expires: 300,
      });
      return { uploadUrl, fileUrl: `https://cdn.example.com/${key}`, expiresIn: 300 };
    },
  },
};
// Option B is preferred: no memory pressure on GraphQL server, faster uploads
```

---

### Q20: Schema Stitching vs Federation.

**Answer:**

```typescript
// Schema Stitching — gateway merges schemas at runtime
import { stitchSchemas } from '@graphql-tools/stitch';
const gateway = stitchSchemas({
  subschemas: [
    { schema: usersSchema, executor: usersExecutor },
    { schema: postsSchema, executor: postsExecutor },
  ],
});
```

| | Schema Stitching | Federation |
|---|---|---|
| Complexity lives in | Gateway | Distributed across services |
| Services know each other | No | Yes (extend types) |
| Type ownership | Gateway decides | Services own their types |
| Tooling | `@graphql-tools/stitch` | Apollo Router |
| Best for | Third-party / legacy API merging | Greenfield microservices |

---

## GraphQL vs REST

### Q21: When to choose REST vs GraphQL?

**Answer:**

| Scenario | Choose | Reason |
|---|---|---|
| Multiple clients needing different shapes | GraphQL | Each client requests exactly what it needs |
| Simple CRUD with fixed responses | REST | Less overhead, native HTTP caching |
| Complex nested data | GraphQL | Single request, no over/under-fetching |
| File upload heavy API | REST | Simpler multipart |
| Public third-party API | REST | More universally understood |
| Rapid frontend iteration | GraphQL | No backend changes needed |
| Real-time + request/response | GraphQL | Subscriptions built-in |
| Microservices aggregation | GraphQL | Federation / gateway |
| Unifying existing REST APIs | GraphQL | Gateway wrapping REST services |

```typescript
// Hybrid: GraphQL gateway over existing REST APIs
const resolvers = {
  Query: {
    user:   (_, { id })    => fetch(`${REST}/users/${id}`).then(r => r.json()),
    trades: (_, { symbol }) => fetch(`${REST}/trades?symbol=${symbol}`).then(r => r.json()),
  },
};
// Frontend gets GraphQL DX; backend keeps REST services untouched
```

---

### Q22: Common GraphQL anti-patterns to avoid.

**Answer:**

```graphql
# ❌ REST-style naming in GraphQL
type Query {
  getUserById(id: ID!): User
  getAllUsers: [User!]!
}
type Mutation {
  createNewUser(input: UserInput): User
}

# ✅ Idiomatic GraphQL
type Query {
  user(id: ID!): User
  users(filter: UserFilter, pagination: PaginationInput): UserConnection!
}
type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
}
```

```typescript
// ❌ Fat resolvers with business logic
const resolvers = {
  Mutation: {
    createOrder: async (_, { input }, ctx) => {
      const user = await db.users.findById(input.userId);
      if (user.balance < input.total) throw new Error('Insufficient funds');
      user.balance -= input.total;
      await user.save();
      const order = await db.orders.create(input);
      await sendEmail(user.email, 'Order confirmed');
      await publishToQueue(order);
      return order;
    },
  },
};

// ✅ Thin resolver — delegate to service layer
const resolvers = {
  Mutation: {
    createOrder: (_, { input }, ctx) => ctx.orderService.create(input),
  },
};
```

```graphql
# ❌ No pagination — crashes at scale
type Query { users: [User!]! }

# ✅ Always paginate
type Query { users(first: Int, after: String): UserConnection! }

# ❌ Leaking DB internals
type User { _id: ObjectId; __v: Int; password_hash: String }

# ✅ Clean API type
type User { id: ID!; name: String!; email: String! }
```

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Previous**: [Behavioral & DSA Q&A](./11-behavioral-dsa.md)
