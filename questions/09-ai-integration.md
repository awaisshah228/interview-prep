# AI Integration - Interview Q&A

> 15+ questions covering LangChain, RAG, AI in production, computer vision, and AI trading

---

## Table of Contents

- [AI Fundamentals for Engineers](#ai-fundamentals-for-engineers)
- [LangChain & RAG](#langchain--rag)
- [AI in Production](#ai-in-production)
- [Computer Vision (YOLO, CVAT)](#computer-vision-yolo-cvat)
- [AI Trading Platforms](#ai-trading-platforms)

---

## AI Fundamentals for Engineers

### Q1: What should a full-stack engineer know about AI/LLMs?

**Answer:**

```
You don't need to train models. You need to know how to:
1. Integrate AI APIs effectively
2. Design good prompts
3. Build RAG systems
4. Handle streaming responses
5. Manage costs and rate limits
6. Evaluate output quality

Key Concepts:
─────────────
Tokens:      ~4 characters = 1 token. You pay per token in/out.
Temperature: 0 = deterministic, 1 = creative. Use 0 for structured output.
Context Window: Max tokens (input + output). GPT-4: 128K, Claude: 200K.
Embeddings:  Convert text to vectors for similarity search.
Fine-tuning: Customize model on your data (expensive, rarely needed).
RAG:         Give the model relevant context at query time (usually better than fine-tuning).
Function Calling: Let the model call your APIs with structured output.
```

---

### Q2: Explain prompt engineering best practices.

**Answer:**

```typescript
// 1. System prompt - define role and constraints
const SYSTEM_PROMPT = `You are a trading assistant for a crypto trading platform.
You analyze market data and provide trade recommendations.

Rules:
- Always include risk assessment (low/medium/high)
- Never provide financial advice - only analysis
- Use exact numbers, not approximations
- Format output as structured JSON

Output format:
{
  "analysis": "string",
  "signals": [{ "symbol": "string", "action": "BUY|SELL|HOLD", "confidence": 0-1 }],
  "risk": "low|medium|high"
}`;

// 2. Few-shot examples (show the model what you want)
const messages = [
  { role: 'system', content: SYSTEM_PROMPT },
  { role: 'user', content: 'SOL dropped 5% in 1 hour, RSI at 28, volume spike 3x' },
  { role: 'assistant', content: JSON.stringify({
    analysis: 'SOL showing oversold conditions with RSI at 28...',
    signals: [{ symbol: 'SOL', action: 'BUY', confidence: 0.72 }],
    risk: 'medium'
  })},
  { role: 'user', content: actualUserQuery },
];

// 3. Chain of Thought (for complex reasoning)
const COT_PROMPT = `Analyze this trade step by step:
1. First, identify the current trend
2. Then, check technical indicators
3. Assess volume patterns
4. Consider market sentiment
5. Finally, provide your recommendation with reasoning`;

// 4. Structured output with JSON mode
const response = await openai.chat.completions.create({
  model: 'gpt-4',
  messages,
  response_format: { type: 'json_object' }, // force JSON output
  temperature: 0,          // deterministic for analysis
  max_tokens: 1000,
});

// 5. Anti-patterns to avoid:
// ❌ Vague prompts: "Tell me about SOL"
// ✅ Specific: "Analyze SOL/USD price action for the last 24h. Include RSI, MACD, volume."
// ❌ No constraints: model writes essays
// ✅ Format constraints: "Respond in JSON with max 3 signals"
```

---

## LangChain & RAG

### Q3: Explain RAG (Retrieval-Augmented Generation) architecture.

**Answer:**

```
RAG = Give the LLM relevant context from YOUR data at query time

Why RAG > Fine-tuning:
✅ No training cost
✅ Data stays up-to-date (just update your docs)
✅ Transparent sources (cite where answer came from)
✅ Works with any LLM
❌ Requires vector database
❌ Latency from retrieval step

Architecture:
═══════════════════════════════════════

INGESTION (one-time or periodic):
┌──────────┐    ┌──────────┐    ┌───────────┐    ┌──────────────┐
│ Documents │───▶│ Chunk    │───▶│ Embed     │───▶│ Vector DB    │
│ (PDF,     │    │ (split   │    │ (OpenAI   │    │ (Pinecone/   │
│  MD, DB)  │    │  ~1000   │    │  ada-002) │    │  Weaviate)   │
└──────────┘    │  tokens)  │    └───────────┘    └──────────────┘
                └──────────┘

QUERY (per user request):
┌──────────┐    ┌───────────┐    ┌──────────────┐
│ Question │───▶│ Embed     │───▶│ Vector       │
└──────────┘    │ Question  │    │ Similarity   │
                └───────────┘    │ Search       │
                                 └──────┬───────┘
                                        │ Top K results
                                        ▼
                                 ┌──────────────┐    ┌──────────┐
                                 │ Prompt:      │───▶│ LLM      │
                                 │ "Given this  │    │ Response │
                                 │  context,    │    └──────────┘
                                 │  answer..."  │
                                 └──────────────┘
```

```typescript
// LangChain RAG implementation
import { ChatOpenAI } from '@langchain/openai';
import { OpenAIEmbeddings } from '@langchain/openai';
import { PineconeStore } from '@langchain/pinecone';
import { RecursiveCharacterTextSplitter } from 'langchain/text_splitter';
import { createRetrievalChain } from 'langchain/chains/retrieval';
import { createStuffDocumentsChain } from 'langchain/chains/combine_documents';

// Step 1: Ingest documents
const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1000,
  chunkOverlap: 200,     // overlap prevents losing context at boundaries
  separators: ['\n\n', '\n', '. ', ' '], // prefer splitting at paragraphs
});

const docs = await splitter.splitDocuments(rawDocuments);

const embeddings = new OpenAIEmbeddings({
  model: 'text-embedding-3-small', // cheaper, fast
});

const vectorStore = await PineconeStore.fromDocuments(docs, embeddings, {
  pineconeIndex,
  namespace: 'trading-docs',
});

// Step 2: Query
const llm = new ChatOpenAI({
  model: 'gpt-4',
  temperature: 0,
});

const retriever = vectorStore.asRetriever({
  k: 5,               // return top 5 matches
  searchType: 'mmr',  // Maximum Marginal Relevance (diversity)
});

const prompt = ChatPromptTemplate.fromMessages([
  ['system', `Answer based on the provided context. If unsure, say "I don't know."

Context: {context}`],
  ['human', '{input}'],
]);

const chain = createRetrievalChain({
  retriever,
  combineDocsChain: await createStuffDocumentsChain({ llm, prompt }),
});

const result = await chain.invoke({
  input: 'What is the maximum position size for SOL trades?',
});

console.log(result.answer);       // AI-generated answer
console.log(result.context);      // Source documents used
```

---

### Q4: Explain LangChain chains and agents.

**Answer:**

```typescript
// CHAINS = fixed sequence of steps

// Simple chain: prompt → LLM → output
import { ChatOpenAI } from '@langchain/openai';
import { StringOutputParser } from '@langchain/core/output_parsers';
import { ChatPromptTemplate } from '@langchain/core/prompts';

const chain = ChatPromptTemplate.fromMessages([
  ['system', 'You are a crypto analyst.'],
  ['human', 'Analyze {symbol} price action'],
])
  .pipe(new ChatOpenAI({ model: 'gpt-4' }))
  .pipe(new StringOutputParser());

const result = await chain.invoke({ symbol: 'SOL' });

// Sequential chain: output of one feeds into next
const analysisChain = prompt1.pipe(llm).pipe(parser);
const summaryChain = prompt2.pipe(llm).pipe(parser);

const fullChain = RunnableSequence.from([
  analysisChain,
  (analysis) => ({ analysis, format: 'executive summary' }),
  summaryChain,
]);

// AGENTS = LLM decides which tools to use dynamically
import { createOpenAIFunctionsAgent, AgentExecutor } from 'langchain/agents';

const tools = [
  new DynamicTool({
    name: 'get_price',
    description: 'Get current price of a cryptocurrency symbol',
    func: async (symbol) => {
      const price = await fetchPrice(symbol);
      return JSON.stringify(price);
    },
  }),
  new DynamicTool({
    name: 'get_trade_history',
    description: 'Get recent trade history for a user',
    func: async (userId) => {
      const trades = await fetchTrades(userId);
      return JSON.stringify(trades);
    },
  }),
  new DynamicTool({
    name: 'execute_trade',
    description: 'Execute a trade. Input: {"symbol": "SOL", "amount": 10, "type": "buy"}',
    func: async (input) => {
      const trade = JSON.parse(input);
      return await executeTrade(trade);
    },
  }),
];

const agent = await createOpenAIFunctionsAgent({ llm, tools, prompt });
const executor = new AgentExecutor({ agent, tools });

// Agent decides which tools to use based on user input
const result = await executor.invoke({
  input: 'Buy 10 SOL if the current price is under $100',
});
// Agent: 1) calls get_price("SOL") → $95
//        2) price < 100, calls execute_trade({"symbol":"SOL","amount":10,"type":"buy"})
//        3) responds: "Bought 10 SOL at $95"
```

---

## AI in Production

### Q5: How do you handle streaming AI responses?

**Answer:**

```typescript
// Backend (NestJS) - Stream from OpenAI to client
@Controller('ai')
export class AIController {
  @Post('analyze')
  @Header('Content-Type', 'text/event-stream')
  @Header('Cache-Control', 'no-cache')
  @Header('Connection', 'keep-alive')
  async streamAnalysis(@Body() dto: AnalyzeDto, @Res() res: Response) {
    const stream = await this.openai.chat.completions.create({
      model: 'gpt-4',
      messages: [
        { role: 'system', content: ANALYST_PROMPT },
        { role: 'user', content: dto.query },
      ],
      stream: true,
    });

    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content || '';
      if (content) {
        res.write(`data: ${JSON.stringify({ content })}\n\n`);
      }
    }

    res.write('data: [DONE]\n\n');
    res.end();
  }
}

// Frontend (React) - Consume SSE stream
function useAIStream() {
  const [response, setResponse] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);

  const analyze = async (query: string) => {
    setIsStreaming(true);
    setResponse('');

    const res = await fetch('/api/ai/analyze', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query }),
    });

    const reader = res.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const text = decoder.decode(value);
      const lines = text.split('\n').filter(line => line.startsWith('data: '));

      for (const line of lines) {
        const data = line.replace('data: ', '');
        if (data === '[DONE]') break;

        const { content } = JSON.parse(data);
        setResponse(prev => prev + content);
      }
    }

    setIsStreaming(false);
  };

  return { response, isStreaming, analyze };
}
```

---

### Q6: AI cost management and optimization.

**Answer:**

```typescript
// Cost-saving strategies for production AI

// 1. Caching (most impactful)
@Injectable()
export class AICacheService {
  constructor(private redis: Redis) {}

  async getCachedOrGenerate(prompt: string, generate: () => Promise<string>) {
    const cacheKey = `ai:${createHash('sha256').update(prompt).digest('hex')}`;
    const cached = await this.redis.get(cacheKey);
    if (cached) return cached;

    const result = await generate();
    await this.redis.setex(cacheKey, 3600, result); // cache 1 hour
    return result;
  }
}

// 2. Model tiering (use cheapest model that works)
function selectModel(task: string): string {
  switch (task) {
    case 'classification':
    case 'extraction':
      return 'gpt-3.5-turbo';    // cheap, fast, good for structured tasks
    case 'analysis':
    case 'reasoning':
      return 'gpt-4';            // expensive but better reasoning
    case 'embedding':
      return 'text-embedding-3-small'; // cheapest embeddings
    default:
      return 'gpt-3.5-turbo';
  }
}

// 3. Token management
function truncateContext(text: string, maxTokens: number): string {
  // Rough estimate: 1 token ≈ 4 chars
  const maxChars = maxTokens * 4;
  if (text.length <= maxChars) return text;
  return text.slice(0, maxChars) + '... [truncated]';
}

// 4. Rate limiting per user
@Injectable()
export class AIRateLimiter {
  async checkLimit(userId: string): Promise<boolean> {
    const key = `ai:usage:${userId}:${getMonth()}`;
    const usage = await this.redis.incr(key);
    if (usage === 1) await this.redis.expire(key, 30 * 24 * 3600);
    return usage <= MAX_MONTHLY_REQUESTS;
  }
}

// 5. Batch processing (cheaper than real-time)
// Queue non-urgent AI tasks, process in batches
// OpenAI Batch API: 50% cheaper, 24-hour SLA

// Cost comparison (per 1M tokens, approximate):
// GPT-4:              $30 input / $60 output
// GPT-4-mini:         $0.15 input / $0.60 output
// GPT-3.5-turbo:      $0.50 input / $1.50 output
// Claude 3 Haiku:     $0.25 input / $1.25 output
// Text-embedding-3-small: $0.02 per 1M tokens
```

---

### Q7: Function calling / tool use with AI.

**Answer:**

```typescript
// Function calling = structured output from AI

const response = await openai.chat.completions.create({
  model: 'gpt-4',
  messages: [
    {
      role: 'user',
      content: 'I want to buy 50 SOL if the price is under $100, and set a stop loss at $90'
    },
  ],
  tools: [
    {
      type: 'function',
      function: {
        name: 'place_order',
        description: 'Place a trading order',
        parameters: {
          type: 'object',
          properties: {
            symbol: { type: 'string', description: 'Trading pair symbol' },
            side: { type: 'string', enum: ['buy', 'sell'] },
            amount: { type: 'number', description: 'Amount to trade' },
            orderType: { type: 'string', enum: ['market', 'limit', 'stop'] },
            price: { type: 'number', description: 'Limit/stop price' },
          },
          required: ['symbol', 'side', 'amount', 'orderType'],
        },
      },
    },
    {
      type: 'function',
      function: {
        name: 'get_price',
        description: 'Get current market price',
        parameters: {
          type: 'object',
          properties: {
            symbol: { type: 'string' },
          },
          required: ['symbol'],
        },
      },
    },
  ],
  tool_choice: 'auto', // let model decide which tools to call
});

// AI returns structured tool calls:
// [
//   { function: { name: 'get_price', arguments: '{"symbol":"SOL"}' } },
//   { function: { name: 'place_order', arguments: '{"symbol":"SOL","side":"buy","amount":50,"orderType":"limit","price":100}' } },
//   { function: { name: 'place_order', arguments: '{"symbol":"SOL","side":"sell","amount":50,"orderType":"stop","price":90}' } },
// ]

// Process tool calls
for (const toolCall of response.choices[0].message.tool_calls) {
  const { name, arguments: args } = toolCall.function;
  const parsedArgs = JSON.parse(args);

  switch (name) {
    case 'get_price':
      return await priceService.getPrice(parsedArgs.symbol);
    case 'place_order':
      return await tradeService.placeOrder(parsedArgs);
  }
}
```

---

## Computer Vision (YOLO, CVAT)

### Q8: Explain CVAT deployment and YOLO integration (your experience).

**Answer:**

```
CVAT (Computer Vision Annotation Tool):
────────────────────────────────────────
Your role: Deployed CVAT on GCP/AWS for annotation workflows

Architecture:
┌──────────────────┐     ┌──────────────┐     ┌──────────────┐
│ Users            │────▶│   CVAT Web   │────▶│   CVAT       │
│ (Annotators)     │     │   (React)    │     │   Backend    │
└──────────────────┘     └──────────────┘     │   (Django)   │
                                               └──────┬───────┘
                                                      │
                              ┌────────────────────────┼─────────┐
                              ▼                        ▼         ▼
                       ┌──────────┐            ┌──────────┐ ┌──────────┐
                       │ Redis    │            │PostgreSQL│ │ S3/GCS   │
                       │ (queue)  │            │(metadata)│ │(images)  │
                       └──────────┘            └──────────┘ └──────────┘
                              │
                              ▼
                       ┌──────────────┐
                       │ YOLO Auto-   │
                       │ Annotation   │
                       │ (Serverless) │
                       └──────────────┘

YOLO Integration workflow:
1. Upload images to CVAT
2. Run YOLO model for auto-annotation (pre-labeling)
3. Human annotators review and correct labels
4. Export annotations (COCO, YOLO, Pascal VOC format)
5. Retrain model with corrected data
6. Deploy updated model

Deployment on AWS:
- ECS Fargate for CVAT services
- S3 for image/video storage
- RDS for PostgreSQL
- ElastiCache for Redis
- ALB for load balancing
- GPU instances (EC2 p3/g4) for YOLO inference
```

```python
# YOLO inference example (Python - serverless function)
from ultralytics import YOLO

model = YOLO('yolov8n.pt')  # nano model for speed

def detect(image_path):
    results = model(image_path)
    detections = []
    for r in results:
        for box in r.boxes:
            detections.append({
                'class': model.names[int(box.cls)],
                'confidence': float(box.conf),
                'bbox': box.xyxy[0].tolist(),  # [x1, y1, x2, y2]
            })
    return detections

# YOLO model sizes:
# YOLOv8n (nano):   3.2M params, fastest, least accurate
# YOLOv8s (small):  11.2M params
# YOLOv8m (medium): 25.9M params
# YOLOv8l (large):  43.7M params
# YOLOv8x (xlarge): 68.2M params, slowest, most accurate
```

---

## AI Trading Platforms

### Q9: How did you build AI-powered trading with auto-strategy execution?

**Answer (for behavioral/technical interviews):**

```
Architecture of AI Trading Platform:
═══════════════════════════════════════

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Market Data  │────▶│ AI Strategy  │────▶│ Trade        │
│ Feed (WSS)   │     │ Engine       │     │ Executor     │
│ - Prices     │     │ - Signal gen │     │ - Risk check │
│ - Volume     │     │ - Backtest   │     │ - Order mgmt │
│ - Order book │     │ - ML models  │     │ - Position   │
└──────────────┘     └──────────────┘     └──────────────┘
                            │                      │
                     ┌──────┴──────┐        ┌──────┴──────┐
                     ▼             ▼        ▼             ▼
              ┌──────────┐  ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Strategy │  │Analytics │ │ Exchange │ │ Notifica-│
              │ Store    │  │Dashboard │ │ APIs     │ │ tions    │
              │ (configs)│  │(Next.js) │ │          │ │ (SNS)    │
              └──────────┘  └──────────┘ └──────────┘ └──────────┘

Components I built:
1. Real-time price feed processing (WebSocket/SSE)
2. AI signal generation (LangChain + custom models)
3. Auto-execution engine with risk management
4. Trading analytics dashboard (Next.js + TradingView Charts)
5. Notification system for trade alerts (SNS + SQS)

Key metrics achieved:
- 20% improvement in transaction speed
- Sub-second strategy execution
- 99.9% uptime for the trading engine

Risk Management:
- Position size limits per strategy
- Stop-loss enforcement
- Daily loss limits
- Circuit breaker on consecutive losses
- Human override capability
```

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [Web3 & Blockchain](./08-web3-blockchain.md) | **Next**: [Security & Auth](./10-security-auth.md)
