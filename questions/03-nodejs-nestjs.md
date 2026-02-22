# Node.js, NestJS & Express - Interview Q&A

> 20+ questions covering Node.js internals, NestJS architecture, Express patterns, and API design

---

## Table of Contents

- [Node.js Core](#nodejs-core)
- [NestJS Framework](#nestjs-framework)
- [Express.js](#expressjs)
- [API Design](#api-design)

---

## Node.js Core

### Q1: Explain the Node.js Event Loop phases.

**Answer:**

```
   ┌───────────────────────────┐
┌─>│           timers          │  ← setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ← deferred I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ← internal use only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  ← retrieve new I/O events, execute callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  ← setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │  ← socket.on('close'), etc.
│  └───────────────────────────┘

Between each phase: process ALL microtasks
Priority: process.nextTick > Promise callbacks > other microtasks
```

```javascript
setTimeout(() => console.log('timeout'), 0);        // timers
setImmediate(() => console.log('immediate'));         // check
process.nextTick(() => console.log('nextTick'));      // microtask (highest)
Promise.resolve().then(() => console.log('promise')); // microtask

// Output: nextTick, promise, timeout/immediate (order of last two varies in main module)

// Inside I/O callback, setImmediate ALWAYS fires before setTimeout:
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate')); // always first here
});
```

---

### Q2: Explain Node.js Streams.

**Answer:**

```javascript
import { createReadStream, createWriteStream } from 'fs';
import { Transform, pipeline } from 'stream';
import { promisify } from 'util';

const pipelineAsync = promisify(pipeline);

// 4 types of streams:
// Readable  - source (fs.createReadStream, http.IncomingMessage, process.stdin)
// Writable  - destination (fs.createWriteStream, http.ServerResponse, process.stdout)
// Transform - read + write + modify (zlib.createGzip, crypto.createCipher)
// Duplex    - read + write independently (net.Socket, WebSocket)

// Transform stream example - process CSV
const csvParser = new Transform({
  objectMode: true,
  transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n').filter(Boolean);
    for (const line of lines) {
      const [symbol, price, volume] = line.split(',');
      this.push({ symbol, price: Number(price), volume: Number(volume) });
    }
    callback();
  },
});

// Pipeline - safely connect streams (handles errors, cleanup)
await pipelineAsync(
  createReadStream('trades.csv'),         // 1. Read file
  csvParser,                               // 2. Parse CSV
  async function* (source) {              // 3. Filter (async generator)
    for await (const trade of source) {
      if (trade.volume > 1000) {
        yield JSON.stringify(trade) + '\n';
      }
    }
  },
  createWriteStream('large-trades.json'), // 4. Write output
);

// Backpressure - automatic flow control
// If writable is slower than readable, the readable pauses automatically
// This prevents memory overflow when processing large files

// Practical: HTTP file upload
app.post('/upload', (req, res) => {
  const writeStream = createWriteStream('/uploads/file.csv');
  req.pipe(writeStream); // req is a Readable, pipe handles backpressure
  writeStream.on('finish', () => res.json({ success: true }));
  writeStream.on('error', (err) => res.status(500).json({ error: err.message }));
});
```

**Why streams matter:**
- Process GBs of data with ~16KB memory (constant buffer)
- Essential for file processing, real-time data, API proxying
- Node.js's core superpower for I/O-heavy workloads

---

### Q3: Worker Threads vs Cluster Module vs Child Processes.

**Answer:**

| Feature | Worker Threads | Cluster | Child Process |
|---|---|---|---|
| Use case | CPU-intensive work | Scale HTTP server | Run external programs |
| Memory | Shared (SharedArrayBuffer) | Separate | Separate |
| Communication | MessagePort (fast) | IPC | IPC / stdin/stdout |
| Overhead | Low | Medium | High |
| Example | Image processing, crypto | Multi-core HTTP | Run Python script |

```javascript
// Worker Threads - CPU-intensive in background
import { Worker, isMainThread, parentPort } from 'worker_threads';

if (isMainThread) {
  // Main thread
  const worker = new Worker('./heavy-task.js');
  worker.postMessage({ data: largeDataset });
  worker.on('message', (result) => console.log('Result:', result));
  worker.on('error', (err) => console.error('Worker error:', err));
} else {
  // Worker thread
  parentPort.on('message', (msg) => {
    const result = expensiveComputation(msg.data);
    parentPort.postMessage(result);
  });
}

// Cluster - scale HTTP across CPU cores
import cluster from 'cluster';
import os from 'os';

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork(); // creates worker process
  }
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died, restarting...`);
    cluster.fork(); // auto-restart
  });
} else {
  // Each worker runs the HTTP server
  app.listen(3000); // OS load-balances connections
}

// Child Process - run external commands
import { exec, spawn } from 'child_process';

// exec: buffered output (small output)
exec('ls -la', (err, stdout, stderr) => {
  console.log(stdout);
});

// spawn: streamed output (large output, long-running)
const child = spawn('python', ['ml-model.py', '--predict']);
child.stdout.on('data', (data) => console.log(data.toString()));
```

---

### Q4: How does Node.js handle errors? Best practices.

**Answer:**

```javascript
// 1. Synchronous errors - try/catch
try {
  JSON.parse('invalid json');
} catch (error) {
  console.error('Parse error:', error.message);
}

// 2. Async errors (callbacks) - error-first callback pattern
fs.readFile('file.txt', (err, data) => {
  if (err) {
    console.error('Read error:', err);
    return;
  }
  // use data
});

// 3. Promise errors - .catch() or try/catch with await
async function fetchData() {
  try {
    const result = await someAsyncOp();
    return result;
  } catch (error) {
    // Handle or re-throw
    throw new AppError('Failed to fetch', { cause: error });
  }
}

// 4. Event emitter errors - MUST handle 'error' event
const emitter = new EventEmitter();
emitter.on('error', (err) => {
  console.error('Emitter error:', err);
});
// If no 'error' listener → process crashes!

// 5. Unhandled rejections and exceptions (safety net)
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection:', reason);
  // Log to monitoring, then exit gracefully
  process.exit(1);
});

process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error);
  // Log, cleanup, exit (DON'T continue, state is unreliable)
  process.exit(1);
});

// 6. Custom error classes
class AppError extends Error {
  constructor(message, options = {}) {
    super(message, { cause: options.cause });
    this.statusCode = options.statusCode || 500;
    this.isOperational = options.isOperational ?? true;
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, { statusCode: 404 });
  }
}

// 7. Graceful shutdown
async function gracefulShutdown(signal) {
  console.log(`${signal} received. Shutting down...`);
  server.close(); // stop accepting new connections
  await db.disconnect();
  await redis.quit();
  process.exit(0);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

---

### Q5: Explain the Buffer and how Node.js handles binary data.

**Answer:**

```javascript
// Buffer = fixed-size chunk of memory (outside V8 heap)
// Used for: file I/O, network, crypto, binary protocols

// Creating buffers
const buf1 = Buffer.from('Hello', 'utf8');        // from string
const buf2 = Buffer.from([0x48, 0x65, 0x6c]);     // from bytes
const buf3 = Buffer.alloc(10);                      // zero-filled, 10 bytes
const buf4 = Buffer.allocUnsafe(10);                // uninitialized (faster, may contain old data)

// Operations
buf1.toString('utf8');       // 'Hello'
buf1.toString('hex');        // '48656c6c6f'
buf1.toString('base64');     // 'SGVsbG8='
buf1.length;                 // 5 (bytes, not chars)
buf1[0];                     // 72 (H in ASCII)

// Concatenate
const combined = Buffer.concat([buf1, buf2]);

// Compare
Buffer.compare(buf1, buf2);  // -1, 0, or 1
buf1.equals(buf2);            // true/false

// Practical: Reading binary protocol header
function parseHeader(buffer) {
  return {
    version: buffer.readUInt8(0),          // 1 byte at offset 0
    messageType: buffer.readUInt16BE(1),   // 2 bytes at offset 1 (big-endian)
    length: buffer.readUInt32BE(3),        // 4 bytes at offset 3
    payload: buffer.subarray(7),           // rest of buffer
  };
}

// Encoding matters for internationalization
const text = '你好';
Buffer.from(text, 'utf8').length;   // 6 bytes (3 per Chinese char)
text.length;                         // 2 characters
```

---

## NestJS Framework

### Q6: NestJS Dependency Injection and Module System.

**Answer:**

```typescript
// NestJS uses an IoC (Inversion of Control) container
// Dependencies are "injected" rather than "created"

// 1. Define a service (Provider)
@Injectable()
export class TradeService {
  constructor(
    @InjectRepository(Trade) private tradeRepo: Repository<Trade>,
    private readonly priceService: PriceService,  // auto-injected by type
    @Inject('REDIS_CLIENT') private redis: Redis, // injected by token
  ) {}

  async executeTrade(dto: CreateTradeDto) {
    const price = await this.priceService.getPrice(dto.symbol);
    const trade = this.tradeRepo.create({ ...dto, price });
    await this.tradeRepo.save(trade);
    await this.redis.publish('trade:executed', JSON.stringify(trade));
    return trade;
  }
}

// 2. Define a module
@Module({
  imports: [
    TypeOrmModule.forFeature([Trade]),
    PriceModule,      // imports PriceService
  ],
  controllers: [TradeController],
  providers: [
    TradeService,
    // Custom providers
    {
      provide: 'REDIS_CLIENT',
      useFactory: async (config: ConfigService) => {
        return new Redis(config.get('REDIS_URL'));
      },
      inject: [ConfigService],
    },
  ],
  exports: [TradeService], // available to modules that import TradeModule
})
export class TradeModule {}

// Provider types:
// useClass     → { provide: TradeService, useClass: MockTradeService }
// useValue     → { provide: 'API_KEY', useValue: 'abc123' }
// useFactory   → { provide: 'DB', useFactory: (config) => createDb(config), inject: [ConfigService] }
// useExisting  → { provide: 'AliasService', useExisting: TradeService }

// Provider scopes:
// DEFAULT    → Singleton (one instance per app)
// REQUEST    → New instance per HTTP request
// TRANSIENT  → New instance every time it's injected

@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

---

### Q7: NestJS Guards, Interceptors, Pipes, Filters - Execution Order.

**Answer:**

```
Request Flow:
Middleware → Guards → Interceptors (before) → Pipes → Route Handler → Interceptors (after) → Exception Filters (on error)
```

```typescript
// GUARD - Auth check (returns true/false)
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);
    if (!token) throw new UnauthorizedException();

    try {
      const payload = this.jwtService.verify(token);
      request.user = payload;
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }

  private extractToken(request: Request): string | undefined {
    return request.headers.authorization?.split(' ')[1];
  }
}

// RBAC Guard
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.includes(user.role);
  }
}

// INTERCEPTOR - Transform response, logging, caching
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const now = Date.now();

    return next.handle().pipe(
      tap(() => {
        this.logger.log(`${method} ${url} - ${Date.now() - now}ms`);
      }),
    );
  }
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, ApiResponse<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<ApiResponse<T>> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}

// PIPE - Validate/transform incoming data
@Injectable()
export class ParseTradeSymbolPipe implements PipeTransform {
  transform(value: string): string {
    const symbol = value.toUpperCase().trim();
    if (!/^[A-Z]{2,10}$/.test(symbol)) {
      throw new BadRequestException(`Invalid symbol: ${value}`);
    }
    return symbol;
  }
}

// EXCEPTION FILTER - Custom error responses
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const status = exception.getStatus();

    response.status(status).json({
      success: false,
      statusCode: status,
      message: exception.message,
      timestamp: new Date().toISOString(),
    });
  }
}

// Apply them
@Controller('trades')
@UseGuards(JwtAuthGuard, RolesGuard)
@UseInterceptors(LoggingInterceptor, TransformInterceptor)
export class TradeController {
  @Post()
  @Roles('trader', 'admin')
  @UsePipes(new ValidationPipe({ whitelist: true }))
  create(@Body() dto: CreateTradeDto) {
    return this.tradeService.create(dto);
  }

  @Get(':symbol')
  findBySymbol(@Param('symbol', ParseTradeSymbolPipe) symbol: string) {
    return this.tradeService.findBySymbol(symbol);
  }
}
```

---

### Q8: NestJS Custom Decorators.

**Answer:**

```typescript
// 1. Parameter decorator - extract user from request
export const CurrentUser = createParamDecorator(
  (data: keyof User | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  },
);

// Usage
@Get('profile')
getProfile(@CurrentUser() user: User) { return user; }

@Get('name')
getName(@CurrentUser('name') name: string) { return name; }

// 2. Method decorator - combine multiple decorators
export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}

// Usage
@Post()
@Auth('admin', 'trader')
create(@Body() dto: CreateTradeDto) { ... }

// 3. Class decorator
export function ApiController(prefix: string) {
  return applyDecorators(
    Controller(prefix),
    ApiTags(prefix),
    UseInterceptors(TransformInterceptor),
    UseFilters(HttpExceptionFilter),
  );
}

@ApiController('trades')
export class TradeController { ... }
```

---

### Q9: NestJS Microservices.

**Answer:**

```typescript
// 1. Create microservice (Trade Service)
// trade-service/main.ts
async function bootstrap() {
  const app = await NestFactory.createMicroservice(TradeModule, {
    transport: Transport.TCP,
    options: { host: '0.0.0.0', port: 3001 },
  });
  await app.listen();
}

// 2. Message patterns in microservice
@Controller()
export class TradeController {
  // Request-Response (sync, returns value)
  @MessagePattern({ cmd: 'get_trade' })
  async getTrade(@Payload() data: { id: string }): Promise<Trade> {
    return this.tradeService.findById(data.id);
  }

  @MessagePattern({ cmd: 'execute_trade' })
  async executeTrade(@Payload() data: CreateTradeDto): Promise<Trade> {
    return this.tradeService.execute(data);
  }

  // Event-based (fire-and-forget, no response)
  @EventPattern('trade_executed')
  async handleTradeExecuted(@Payload() data: TradeEvent): Promise<void> {
    await this.analyticsService.recordTrade(data);
  }
}

// 3. API Gateway (client)
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'TRADE_SERVICE',
        transport: Transport.TCP,
        options: { host: 'trade-service', port: 3001 },
      },
      {
        name: 'NOTIFICATION_SERVICE',
        transport: Transport.REDIS,
        options: { host: 'redis', port: 6379 },
      },
    ]),
  ],
})
export class GatewayModule {}

@Controller('trades')
export class GatewayController {
  constructor(
    @Inject('TRADE_SERVICE') private tradeClient: ClientProxy,
    @Inject('NOTIFICATION_SERVICE') private notifClient: ClientProxy,
  ) {}

  @Get(':id')
  getTrade(@Param('id') id: string) {
    // send() = request-response
    return this.tradeClient.send({ cmd: 'get_trade' }, { id });
  }

  @Post()
  async createTrade(@Body() dto: CreateTradeDto) {
    const trade = await firstValueFrom(
      this.tradeClient.send({ cmd: 'execute_trade' }, dto)
    );
    // emit() = event (fire and forget)
    this.notifClient.emit('trade_executed', trade);
    return trade;
  }
}

// Transport options:
// TCP     - simple, internal services
// Redis   - pub/sub, good for events
// NATS    - lightweight, high performance
// Kafka   - high throughput, event streaming
// gRPC    - binary protocol, schema validation
// RabbitMQ - advanced routing, guaranteed delivery
```

---

### Q10: NestJS WebSockets and SSE.

**Answer:**

```typescript
// WebSocket Gateway
@WebSocketGateway({
  cors: { origin: '*' },
  namespace: '/trades',
})
export class TradeGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);
  }

  // Listen for client events
  @SubscribeMessage('subscribe_symbol')
  handleSubscribe(client: Socket, symbol: string) {
    client.join(`symbol:${symbol}`);
    return { event: 'subscribed', data: symbol };
  }

  // Broadcast price update to subscribers
  broadcastPrice(symbol: string, price: number) {
    this.server.to(`symbol:${symbol}`).emit('price_update', { symbol, price });
  }
}

// SSE (Server-Sent Events) - simpler, one-directional
@Controller('events')
export class EventController {
  constructor(private tradeService: TradeService) {}

  @Sse('trades')
  tradeEvents(): Observable<MessageEvent> {
    return this.tradeService.tradeStream$.pipe(
      map(trade => ({
        data: JSON.stringify(trade),
        type: 'trade',
        id: trade.id,
      })),
    );
  }
}

// Client-side SSE
const eventSource = new EventSource('/events/trades');
eventSource.addEventListener('trade', (event) => {
  const trade = JSON.parse(event.data);
  updateUI(trade);
});

// WebSocket vs SSE:
// WebSocket: bidirectional, binary support, more complex
// SSE: server→client only, auto-reconnect, simpler, HTTP/2 compatible
// Use WebSocket for: chat, gaming, real-time collaboration
// Use SSE for: price feeds, notifications, live dashboards
```

---

### Q11: NestJS with TypeORM / Prisma.

**Answer:**

```typescript
// TypeORM Entity
@Entity('trades')
export class Trade {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  symbol: string;

  @Column('decimal', { precision: 18, scale: 8 })
  amount: number;

  @Column('decimal', { precision: 18, scale: 8 })
  price: number;

  @Column({ type: 'enum', enum: TradeType })
  type: TradeType;

  @ManyToOne(() => User, user => user.trades)
  @JoinColumn({ name: 'user_id' })
  user: User;

  @CreateDateColumn()
  createdAt: Date;

  @Index()
  @Column()
  executedAt: Date;
}

// Repository pattern
@Injectable()
export class TradeService {
  constructor(
    @InjectRepository(Trade) private repo: Repository<Trade>,
  ) {}

  async findUserTrades(userId: string, options: PaginationDto) {
    return this.repo.find({
      where: { user: { id: userId } },
      order: { executedAt: 'DESC' },
      take: options.limit,
      skip: options.offset,
      relations: ['user'],
    });
  }

  async getDailyPnL(userId: string) {
    return this.repo
      .createQueryBuilder('t')
      .select("DATE_TRUNC('day', t.executed_at)", 'date')
      .addSelect('SUM(t.pnl)', 'dailyPnl')
      .addSelect('COUNT(*)', 'tradeCount')
      .where('t.user_id = :userId', { userId })
      .groupBy("DATE_TRUNC('day', t.executed_at)")
      .orderBy('date', 'DESC')
      .limit(30)
      .getRawMany();
  }
}

// Prisma alternative (schema-driven)
// prisma/schema.prisma
// model Trade {
//   id        String   @id @default(uuid())
//   symbol    String
//   amount    Decimal
//   price     Decimal
//   type      TradeType
//   userId    String   @map("user_id")
//   user      User     @relation(fields: [userId], references: [id])
//   createdAt DateTime @default(now()) @map("created_at")
//   @@index([userId, createdAt])
//   @@map("trades")
// }

// Prisma usage in NestJS
@Injectable()
export class TradeService {
  constructor(private prisma: PrismaService) {}

  async findUserTrades(userId: string) {
    return this.prisma.trade.findMany({
      where: { userId },
      orderBy: { createdAt: 'desc' },
      include: { user: true },
    });
  }
}
```

---

## Express.js

### Q12: Express Middleware Chain - How does it work?

**Answer:**

```javascript
// Middleware = function(req, res, next)
// next() passes control to next middleware
// Not calling next() stops the chain

const express = require('express');
const app = express();

// 1. Application-level middleware (runs on every request)
app.use((req, res, next) => {
  req.startTime = Date.now();
  console.log(`${req.method} ${req.path}`);
  next(); // pass to next middleware
});

// 2. Built-in middleware
app.use(express.json());                    // parse JSON body
app.use(express.urlencoded({ extended: true })); // parse form data
app.use(express.static('public'));          // serve static files

// 3. Third-party middleware
app.use(cors());
app.use(helmet());        // security headers
app.use(compression());   // gzip responses
app.use(morgan('dev'));   // request logging

// 4. Router-level middleware
const router = express.Router();
router.use(authMiddleware); // only for this router's routes
router.get('/trades', getTradesHandler);
app.use('/api', router);

// 5. Error-handling middleware (4 params!)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.statusCode || 500).json({
    error: err.message || 'Internal Server Error',
  });
});

// Middleware execution order:
// 1. app.use (in order defined)
// 2. Route-specific middleware
// 3. Route handler
// 4. Error middleware (if error thrown or next(error) called)

// Async error handling (Express doesn't catch async errors by default)
// Option 1: Wrapper
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/trades', asyncHandler(async (req, res) => {
  const trades = await Trade.find(); // if this throws, error middleware catches it
  res.json(trades);
}));

// Option 2: express-async-errors (monkey-patches Express)
require('express-async-errors');
// Now async errors are automatically caught
```

---

### Q13: Express Security Best Practices.

**Answer:**

```javascript
const express = require('express');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const cors = require('cors');
const hpp = require('hpp');
const mongoSanitize = require('express-mongo-sanitize');

const app = express();

// 1. Helmet - sets security headers
app.use(helmet());
// Sets: X-Content-Type-Options, X-Frame-Options, X-XSS-Protection,
//       Content-Security-Policy, Strict-Transport-Security, etc.

// 2. CORS - restrict origins
app.use(cors({
  origin: ['https://myapp.com', 'https://admin.myapp.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true,
}));

// 3. Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                   // 100 requests per window
  message: 'Too many requests',
  standardHeaders: true,
});
app.use('/api/', limiter);

// Stricter for auth endpoints
const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 5,                    // 5 login attempts
});
app.use('/api/auth/login', authLimiter);

// 4. Body size limit (prevent large payload attacks)
app.use(express.json({ limit: '10kb' }));

// 5. HTTP Parameter Pollution
app.use(hpp());

// 6. NoSQL injection prevention
app.use(mongoSanitize()); // removes $ and . from req.body/query

// 7. Input validation (use class-validator or Joi)
// Always validate and sanitize user input

// 8. Don't expose stack traces in production
if (process.env.NODE_ENV === 'production') {
  app.use((err, req, res, next) => {
    res.status(err.statusCode || 500).json({
      message: 'Something went wrong', // generic message
    });
  });
}
```

---

## API Design

### Q14: REST API Design Best Practices.

**Answer:**

```
URL Design:
GET    /api/v1/trades           → List trades (with pagination)
GET    /api/v1/trades/:id       → Get single trade
POST   /api/v1/trades           → Create trade
PUT    /api/v1/trades/:id       → Full update
PATCH  /api/v1/trades/:id       → Partial update
DELETE /api/v1/trades/:id       → Delete trade

Filtering, Sorting, Pagination:
GET /api/v1/trades?symbol=SOL&type=buy&sort=-executedAt&page=2&limit=20

Status Codes:
200 OK              → Successful GET/PUT/PATCH
201 Created         → Successful POST
204 No Content      → Successful DELETE
400 Bad Request     → Validation error
401 Unauthorized    → Missing/invalid auth
403 Forbidden       → Authenticated but not authorized
404 Not Found       → Resource doesn't exist
409 Conflict        → Duplicate or state conflict
422 Unprocessable   → Semantic validation error
429 Too Many Requests → Rate limited
500 Internal Error  → Server bug
```

```typescript
// Response format (consistent)
interface ApiResponse<T> {
  success: boolean;
  data: T;
  meta?: {
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
  };
  error?: {
    code: string;
    message: string;
    details?: Record<string, string[]>;
  };
}

// Pagination types:
// 1. Offset-based: ?page=2&limit=20
//    Pro: simple, can jump to any page
//    Con: inconsistent with inserts/deletes, slow on large offsets

// 2. Cursor-based: ?cursor=abc123&limit=20
//    Pro: consistent, performant on large datasets
//    Con: can't jump to specific page

// Versioning strategies:
// URL:    /api/v1/trades (most common, your approach)
// Header: Accept: application/vnd.myapp.v1+json
// Query:  /api/trades?version=1
```

---

### Q15: gRPC vs REST vs GraphQL - When to use each?

**Answer:**

| Feature | REST | GraphQL | gRPC |
|---|---|---|---|
| Protocol | HTTP/1.1 or 2 | HTTP | HTTP/2 |
| Data format | JSON | JSON | Protocol Buffers (binary) |
| Contract | OpenAPI/Swagger | Schema (SDL) | Proto files |
| Over-fetching | Common | Solved | N/A |
| Under-fetching | Common | Solved | N/A |
| Streaming | Limited (SSE) | Subscriptions | Bidirectional |
| Performance | Good | Good | Excellent |
| Browser support | Native | Native | Needs proxy |
| Best for | Public APIs | Frontend-driven | Microservices |

```
When to use:
REST:
  ✅ Public APIs, CRUD operations
  ✅ Caching (HTTP cache-friendly)
  ✅ Simple, well-understood
  Your usage: External APIs, webhook endpoints

GraphQL:
  ✅ Complex, nested data requirements
  ✅ Multiple client types (mobile, web, TV)
  ✅ Reduce over-fetching/under-fetching
  ✅ Real-time with subscriptions

gRPC:
  ✅ Microservice-to-microservice communication
  ✅ High performance, low latency
  ✅ Bidirectional streaming
  ✅ Strong typing with proto files
  Your usage: Geyser gRPC for Solana indexing, internal services
```

---

### Q16: How do you handle API rate limiting and throttling?

**Answer:**

```typescript
// NestJS Throttler
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,   // 1 second window
        limit: 3,     // 3 requests
      },
      {
        name: 'medium',
        ttl: 10000,  // 10 second window
        limit: 20,    // 20 requests
      },
      {
        name: 'long',
        ttl: 60000,  // 1 minute window
        limit: 100,   // 100 requests
      },
    ]),
  ],
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule {}

// Custom throttle per endpoint
@Controller('trades')
export class TradeController {
  @Post()
  @Throttle({ short: { limit: 1, ttl: 1000 } }) // 1 trade per second
  create() { ... }

  @Get()
  @SkipThrottle() // no rate limit for reads
  findAll() { ... }
}

// Redis-based rate limiting (distributed, for multiple server instances)
@Injectable()
export class RedisThrottlerStorage implements ThrottlerStorage {
  constructor(private redis: Redis) {}

  async increment(key: string, ttl: number): Promise<ThrottlerStorageRecord> {
    const multi = this.redis.multi();
    multi.incr(key);
    multi.pttl(key);
    const [[, count], [, ttlResult]] = await multi.exec();

    if (ttlResult === -1) {
      await this.redis.pexpire(key, ttl);
    }

    return { totalHits: count, timeToExpire: ttlResult > 0 ? ttlResult : ttl };
  }
}

// Algorithms:
// Fixed Window    → simple, but burst at window edges
// Sliding Window  → more accurate, prevents edge bursts
// Token Bucket    → allows bursts up to bucket size, then steady rate
// Leaky Bucket    → constant output rate, excess queued/dropped
```

---

## References & Deep Dive Resources

### Node.js Core
| Topic | Resource |
|---|---|
| Node.js Event Loop | [Node.js Event Loop (Official)](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) |
| Event Loop Visualized | [Lydia Hallie - JavaScript Visualized: Event Loop](https://dev.to/lydiahallie/javascript-visualized-event-loop-3dif) |
| Event Loop Video | [Event Loop Visualized (YouTube)](https://www.youtube.com/watch?v=eiC58R16hb8&t=540s) |
| Node.js Streams | [Node.js Streams Handbook](https://nodejs.org/api/stream.html) |
| Streams Guide | [Node.js Streams: Everything You Need to Know](https://www.freecodecamp.org/news/node-js-streams-everything-you-need-to-know-c9141306be93/) |
| Worker Threads | [Node.js Worker Threads](https://nodejs.org/api/worker_threads.html) |
| Cluster Module | [Node.js Cluster](https://nodejs.org/api/cluster.html) |
| Error Handling | [Node.js Error Handling Best Practices](https://nodejs.org/en/learn/getting-started/nodejs-the-difference-between-development-and-production) |
| Buffer | [Node.js Buffer API](https://nodejs.org/api/buffer.html) |
| Node.js Best Practices | [goldbergyoni/nodebestpractices](https://github.com/goldbergyoni/nodebestpractices) — 100+ best practices |

### NestJS
| Topic | Resource |
|---|---|
| NestJS Docs (Official) | [docs.nestjs.com](https://docs.nestjs.com/) |
| Modules & DI | [NestJS - Modules](https://docs.nestjs.com/modules) |
| Guards | [NestJS - Guards](https://docs.nestjs.com/guards) |
| Interceptors | [NestJS - Interceptors](https://docs.nestjs.com/interceptors) |
| Pipes | [NestJS - Pipes](https://docs.nestjs.com/pipes) |
| Exception Filters | [NestJS - Exception Filters](https://docs.nestjs.com/exception-filters) |
| Custom Decorators | [NestJS - Custom Decorators](https://docs.nestjs.com/custom-decorators) |
| Microservices | [NestJS - Microservices](https://docs.nestjs.com/microservices/basics) |
| WebSockets | [NestJS - WebSockets](https://docs.nestjs.com/websockets/gateways) |
| SSE | [NestJS - Server-Sent Events](https://docs.nestjs.com/techniques/server-sent-events) |
| CQRS | [NestJS - CQRS](https://docs.nestjs.com/recipes/cqrs) |
| Execution Context | [NestJS - Execution Context](https://docs.nestjs.com/fundamentals/execution-context) |
| NestJS Course (Free) | [NestJS Crash Course (Traversy)](https://www.youtube.com/watch?v=2n3xS89TJMI) |

### Express
| Topic | Resource |
|---|---|
| Express Guide | [expressjs.com](https://expressjs.com/en/guide/routing.html) |
| Middleware | [Express - Using Middleware](https://expressjs.com/en/guide/using-middleware.html) |
| Error Handling | [Express - Error Handling](https://expressjs.com/en/guide/error-handling.html) |
| Security (helmet) | [helmetjs.github.io](https://helmetjs.github.io/) |

### API Design
| Topic | Resource |
|---|---|
| REST API Design | [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) |
| REST Best Practices | [restfulapi.net](https://restfulapi.net/) |
| GraphQL | [graphql.org/learn](https://graphql.org/learn/) |
| gRPC | [grpc.io/docs](https://grpc.io/docs/) |
| API Pagination | [Slack API - Pagination](https://api.slack.com/docs/pagination) — Great real-world example |
| Rate Limiting Algorithms | [Cloudflare Blog - Rate Limiting](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) |

### ORM / Database Access
| Topic | Resource |
|---|---|
| TypeORM Docs | [typeorm.io](https://typeorm.io/) |
| Prisma Docs | [prisma.io/docs](https://www.prisma.io/docs/) |
| Mongoose Docs | [mongoosejs.com](https://mongoosejs.com/docs/) |

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [React & Next.js](./02-react-nextjs.md) | **Next**: [Databases](./04-databases.md)
