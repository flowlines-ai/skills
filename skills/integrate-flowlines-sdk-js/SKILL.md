---
name: integrate-flowlines-sdk-js
description: Integrates the @flowlines/sdk into a Node.js/TypeScript codebase. Use when adding Flowlines observability to an AI/LLM application, setting up OpenTelemetry instrumentation for AI libraries, or when the user asks to add Flowlines monitoring.
---

# Flowlines SDK Integration Guide

This document helps AI coding agents integrate the `@flowlines/sdk` into a Node.js/TypeScript codebase.

## What is the Flowlines SDK?

`@flowlines/sdk` is a TypeScript SDK that provides OpenTelemetry-based observability for AI/LLM applications. It automatically instruments 13+ AI libraries (OpenAI, Anthropic, Cohere, AWS Bedrock, Google Vertex AI, LangChain, etc.), filters to only export LLM-relevant spans, and sends them to the Flowlines backend.

## Installation

```bash
npm install @flowlines/sdk
```

- Requires Node.js >= 20
- Requires ESM (`"type": "module"` in package.json)

## Public API

The SDK exports a single static class:

```typescript
import { Flowlines } from "@flowlines/sdk";
```

| Export | Kind | Purpose |
|---|---|---|
| `Flowlines` | Static class | Main entry point. Provides `init()`, `shutdown()`, `context()`, `getMemory()`, `endSession()`, `createSpanProcessor()`, and `getInstrumentations()` static methods. |

## Configuration

```typescript
Flowlines.init({
  apiKey: string;                        // Required. Flowlines API key.
  ingestEndpoint?: string;               // Default: "https://ingest.flowlines.ai"
  apiEndpoint?: string;                  // Default: "https://api.flowlines.ai" (used by getMemory, endSession)
  hasExternalOtel?: boolean;             // Default: false. Set true to compose with your own NodeSDK.
  instrumentModules?: InstrumentModules; // Default: undefined (all libraries instrumented).
  exportIntervalMs?: number;             // Default: 5000
  maxQueueSize?: number;                 // Default: 2048
  maxExportBatchSize?: number;           // Default: 512
  verbose?: boolean;                     // Default: false
});
```

**Validation rules:**
- `apiKey` must be a non-empty string
- `ingestEndpoint` and `apiEndpoint` must use HTTPS (HTTP is only allowed for localhost/127.0.0.1)

## Two Integration Modes

### 1. Standalone Mode (recommended for most apps)

Flowlines creates and manages its own OpenTelemetry `NodeSDK`. This is the simplest setup.

```typescript
import OpenAI from "openai";
import { Flowlines } from "@flowlines/sdk";

// CRITICAL: Initialize BEFORE creating any AI client instances.
// Use instrumentModules to pass the imported module for reliable ESM instrumentation.
Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
  instrumentModules: { openAI: OpenAI },
});

// Now create your AI clients — they will be automatically instrumented.
const openai = new OpenAI();

// Use Flowlines.context to attach user/session metadata to spans.
const response = await Flowlines.context({ userId: "user-123", sessionId: "sess-456" }, () =>
  openai.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: "Hello" }],
  })
);

// Shutdown gracefully before process exit.
await Flowlines.shutdown();
```

> **ESM note:** In ESM applications, all `import` statements are hoisted to the top of the module and executed before any other code. This means auto-instrumentation (without `instrumentModules`) may not patch modules in time. Always use `instrumentModules` to pass the imported modules explicitly — this guarantees reliable instrumentation regardless of import order.

### 2. External OpenTelemetry Mode

Use when the app already has its own `NodeSDK` setup and you want to add Flowlines to the existing pipeline.

```typescript
import { Flowlines } from "@flowlines/sdk";
import { NodeSDK } from "@opentelemetry/sdk-node";

Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
  hasExternalOtel: true,
});

const sdk = new NodeSDK({
  spanProcessors: [Flowlines.createSpanProcessor()],
  instrumentations: Flowlines.getInstrumentations(),
});
sdk.start();

// ... use Flowlines.context as usual ...

await Flowlines.shutdown();
await sdk.shutdown();
```

## Attaching context to spans

`Flowlines.context` attaches `flowlines.user_id`, `flowlines.session_id`, and `flowlines.agent_id` attributes to all LLM spans created within its callback. This is how Flowlines associates traces with users, sessions, and agents.

```typescript
interface FlowlinesContext {
  userId: string;
  sessionId: string;
  agentId?: string;
}

Flowlines.context<T>(ctx: FlowlinesContext, fn: () => T): T;
```

- `userId` is required (string)
- `sessionId` is required (string)
- `agentId` is optional (omit to skip)
- `fn` can be sync or async — the return value (or Promise) is forwarded
- Supports nesting: inner calls override outer values
- Must wrap every AI/LLM call that should carry user context

```typescript
// Async usage with session and agent
const result = await Flowlines.context(
  { userId: "user-123", sessionId: "sess-456", agentId: "agent-1" },
  async () => {
    return await openai.chat.completions.create({ /* ... */ });
  }
);

// Without agent ID
await Flowlines.context({ userId: "user-123", sessionId: "sess-789" }, async () => {
  await anthropic.messages.create({ /* ... */ });
});

// Nested contexts (inner overrides outer)
await Flowlines.context({ userId: "user-A", sessionId: "sess-1" }, async () => {
  // spans here have user_id="user-A", session_id="sess-1"
  await Flowlines.context({ userId: "user-B", sessionId: "sess-2" }, async () => {
    // spans here have user_id="user-B", session_id="sess-2"
  });
});
```

## Context integration guidance

When integrating `Flowlines.context`, you MUST wrap LLM calls with context. Follow these steps:

1. **Identify existing data** in the codebase that maps to `userId`, `sessionId`, and `agentId`:
   - `userId`: the end-user making the request (e.g., authenticated user ID, email, API key owner)
   - `sessionId` (required): the conversation or session grouping multiple interactions (e.g., chat thread ID, session token, conversation UUID)
   - `agentId` (optional): the AI agent or assistant handling the request (e.g., agent name, bot identifier, assistant ID)

2. **If obvious mappings exist**, use them directly. For example, if the app has `req.user.id` and a `threadId`, wire them in:
   ```typescript
   await Flowlines.context({ userId: req.user.id, sessionId: threadId }, async () => { ... });
   ```

3. **If mappings are unclear**, ask the user which variables or fields should be used for `userId`, `sessionId`, and `agentId`.

4. **If no data is available yet**, propose using placeholder values with TODO comments so the integration is functional and easy to complete later:
   ```typescript
   await Flowlines.context(
     {
       userId: "anonymous", // TODO: replace with actual user identifier
       sessionId: `sess-${Date.now()}`, // TODO: replace with actual session/conversation ID
       agentId: "my-agent", // TODO: replace with actual agent identifier
     },
     async () => { ... }
   );
   ```
   Only include optional fields that are relevant. `agentId` can be omitted entirely if not applicable.

## Fetching memory

Use `Flowlines.getMemory()` to retrieve memory context from the Flowlines backend for a given user:

```typescript
import { Flowlines } from "@flowlines/sdk";

const memory = await Flowlines.getMemory({
  userId: "user_123",
  sessionId: "sess_456",  // optional
  agentId: "agent_1",     // optional
  view: "summary",        // optional
});
```

Returns a JSON string with the memory content, or an empty string if no memory is found.

## Ending a session

Use `Flowlines.endSession()` to mark a session as ended on the Flowlines backend. This flushes all pending spans before sending the end-session request, ensuring the backend has received all traces for the session.

```typescript
import { Flowlines } from "@flowlines/sdk";

await Flowlines.endSession({
  userId: "user_123",
  sessionId: "sess_456",
});
```

## Selective Instrumentation with `instrumentModules`

By default, all 13+ supported libraries are instrumented. To instrument only specific libraries, pass `instrumentModules` with the imported modules. **The required import style varies by library:**

**OpenAI — use the default import:**

```typescript
import OpenAI from "openai";

Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
  instrumentModules: { openAI: OpenAI },
});
```

**Anthropic — use a namespace import:**

```typescript
import * as AnthropicModule from "@anthropic-ai/sdk";

Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
  instrumentModules: { anthropic: AnthropicModule },
});
```

**Both together:**

```typescript
import OpenAI from "openai";
import * as AnthropicModule from "@anthropic-ai/sdk";

Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
  instrumentModules: {
    openAI: OpenAI,
    anthropic: AnthropicModule,
  },
});
```

Supported keys: `openAI`, `anthropic`, `cohere`, `bedrock`, `google_vertexai`, `google_aiplatform`, `google_generativeai`, `pinecone`, `together`, `chromadb`, `qdrant`, `langchain`, `llamaIndex`, `mcp`.

**Important:** The import style required depends on the library. OpenAI requires the **default import** (the class itself). Anthropic and most other libraries require a **namespace import** (`import * as X from "..."`). Using the wrong import style will cause instrumentation to silently fail or throw errors.

## Critical Integration Rules

1. **Initialize Flowlines BEFORE creating AI client instances.** The SDK uses monkey-patching via OpenTelemetry instrumentation. Clients created before `Flowlines.init()` will not be instrumented.

2. **Only one Flowlines instance can exist at a time.** Calling `Flowlines.init()` when already initialized is a no-op (the call is silently skipped). Call `Flowlines.shutdown()` first if you need to re-initialize with different settings.

3. **Always call `Flowlines.shutdown()` before process exit.** This flushes any pending spans to the backend. Without it, the last batch of traces may be lost.

4. **Wrap LLM calls with `Flowlines.context()` to associate traces with users.** Without `Flowlines.context()`, spans are still exported but lack user/session/agent metadata.

5. **The SDK only exports LLM-related spans.** It filters for spans with `gen_ai.*` or `ai.*` attribute prefixes. Non-LLM OpenTelemetry spans (HTTP, DB, etc.) are discarded by the Flowlines exporter.

## Graceful Shutdown Pattern

```typescript
Flowlines.init({ apiKey: "..." });

// Handle process signals
process.on("SIGINT", async () => {
  await Flowlines.shutdown();
  process.exit(0);
});

process.on("SIGTERM", async () => {
  await Flowlines.shutdown();
  process.exit(0);
});
```

## Known Limitations

- **OpenAI `responses.create` is not instrumented.** The underlying OpenTelemetry instrumentation only patches `chat.completions.create`. Calls to the newer `responses.create` API will not produce traces. Use `chat.completions.create` for traced interactions.

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `Flowlines API key is required` | Empty or missing `apiKey` | Provide a valid API key |
| `Cannot set both hasExternalOtel and hasTraceloop` | Both flags set to true | Use only one mode flag |
| `Invalid Flowlines endpoint URL` | Malformed URL in `ingestEndpoint` or `apiEndpoint` | Provide a valid URL |
| `Insecure Flowlines endpoint` | HTTP endpoint that is not localhost | Use HTTPS, or use localhost for development |

## Full Example: Anthropic Chat with Tools

```typescript
import * as AnthropicModule from "@anthropic-ai/sdk";
import Anthropic from "@anthropic-ai/sdk";
import { Flowlines } from "@flowlines/sdk";

// 1. Init Flowlines FIRST
Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
  instrumentModules: { anthropic: AnthropicModule },
});

// 2. Then create the AI client
const client = new Anthropic();

// 3. Wrap calls with Flowlines.context
const sessionId = `sess-${Date.now()}`;
const reply = await Flowlines.context({ userId: "user-123", sessionId }, () =>
  client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello!" }],
  })
);

// 4. Shutdown before exit
await Flowlines.shutdown();
```
