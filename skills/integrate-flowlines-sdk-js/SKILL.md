---
name: integrate-flowlines-sdk-js
description: Integrates the @flowlines/sdk into a Node.js/TypeScript/Next.js codebase. Use when adding Flowlines observability and memory to an AI/LLM application, setting up OpenTelemetry instrumentation for AI libraries, or when the user asks to add Flowlines monitoring or memory.
---

# Flowlines SDK Integration Guide

Follow these steps to integrate the `@flowlines/sdk` into a Node.js/TypeScript application.

## Step 1: Install the SDK

```bash
npm install @flowlines/sdk
```

Requires Node.js >= 20.

## Step 2: Initialize the SDK

The SDK instruments AI libraries by monkey-patching their module exports. This means `Flowlines.init()` must run **before** those libraries are imported and before any AI client instances are created.

Choose the initialization approach based on the application's setup:

### Standard Node.js app

**Option A: Separate instrumentation file (recommended)**

Create a dedicated file that runs before the application entry point:

```typescript
// instrumentation.ts
import { Flowlines } from "@flowlines/sdk";

Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
});
```

Start the app with:

```bash
# ESM
node --import ./instrumentation.ts app.ts

# CJS
node --require ./instrumentation.js app.js
```

This ensures init runs before any AI library imports.

**Option B: Pass `instrumentModules` explicitly**

If you cannot control the startup order (e.g., ESM hoists all imports), pass the AI modules directly — this works regardless of import order:

```typescript
import OpenAI from "openai";
import { Flowlines } from "@flowlines/sdk";

Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
  instrumentModules: { openAI: OpenAI },
});

const openai = new OpenAI();
```

Supported `instrumentModules` keys: `openAI`, `anthropic`, `cohere`, `bedrock`, `google_vertexai`, `google_aiplatform`, `google_generativeai`, `pinecone`, `together`, `chromadb`, `qdrant`, `langchain`, `llamaIndex`, `mcp`.

> **Import style matters.** OpenAI requires a **default import** (`import OpenAI from "openai"`). Anthropic requires a **namespace import** (`import * as AnthropicModule from "@anthropic-ai/sdk"`). Using the wrong style causes instrumentation to silently fail.

### Next.js app

Next.js supports an [instrumentation file](https://nextjs.org/docs/app/guides/instrumentation) that runs before the application starts.

**1. Create the instrumentation file:**

```typescript
// instrumentation.ts (project root)
import { Flowlines } from "@flowlines/sdk";

export function register() {
  Flowlines.init({
    apiKey: process.env.FLOWLINES_API_KEY,
  });
}
```

**2. Configure webpack externals** in `next.config.mjs` to prevent bundling the SDK and AI libraries (required for monkey-patching to work):

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    instrumentationHook: true,
  },
  webpack: (config, { isServer }) => {
    if (isServer) {
      config.externals.push("@flowlines/sdk", "openai");
    }
    return config;
  },
};

export default nextConfig;
```

Add every instrumented AI library to the `externals` array (e.g. `"openai"`, `"@anthropic-ai/sdk"`).

If auto-instrumentation still doesn't work despite externals, use `instrumentModules` in the `register` function (see Option B above).

### App with existing OpenTelemetry setup

If the app already has its own `NodeSDK`, set `hasExternalOtel: true` so Flowlines does not create a second one. Then compose Flowlines' processor and instrumentations into the existing setup:

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { Flowlines } from "@flowlines/sdk";

Flowlines.init({
  apiKey: process.env.FLOWLINES_API_KEY,
  hasExternalOtel: true,
});

const sdk = new NodeSDK({
  spanProcessors: [Flowlines.createSpanProcessor()],
  instrumentations: Flowlines.getInstrumentations(),
});
sdk.start();
```

## Step 3: Attach context to LLM calls

Wrap every AI/LLM call with `Flowlines.context()` to attach user and session metadata to spans:

```typescript
await Flowlines.context({ userId: "user-123", sessionId: "sess-456" }, async () => {
  const response = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [{ role: "user", content: "Hello!" }],
  });
});
```

- `userId` (required): the end-user making the request
- `sessionId` (required): the conversation or session ID
- `agentId` (optional): the AI agent handling the request
- The callback can be sync or async — the return value is forwarded
- Supports nesting: inner calls override outer values

**How to choose values:**

1. If the codebase has obvious mappings (e.g. `req.user.id`, `threadId`), use them directly.
2. If mappings are unclear, ask the user which variables to use.
3. If no data is available yet, use placeholder values with TODO comments:

```typescript
await Flowlines.context(
  {
    userId: "anonymous", // TODO: replace with actual user identifier
    sessionId: `sess-${Date.now()}`, // TODO: replace with actual session/conversation ID
  },
  async () => { ... }
);
```

## Step 4: Fetch and use memory

Use `Flowlines.getMemory()` to retrieve memory context for a user. Inject the returned string into your LLM conversation (e.g. as a system message):

```typescript
const memory = await Flowlines.getMemory({
  userId: "user_123",
  sessionId: "sess_456",
  agentId: "agent_1",     // optional
});

await Flowlines.context({ userId: "user_123", sessionId: "sess_456" }, async () => {
  const response = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [
      { role: "system", content: `You are a helpful assistant.\n\n${memory}` },
      { role: "user", content: "What were we talking about last time?" },
    ],
  });
});
```

`getMemory()` returns a JSON string with the memory content, or an empty string if no memory is found.

## Step 5: End a session

Look for places where a session or conversation naturally ends and call `Flowlines.endSession()` there. This flushes pending spans and notifies the backend. Common examples:

- An explicit "end conversation" or "close session" action
- A WebSocket `close` or `disconnect` event
- A session cleanup or logout handler
- A timeout that expires inactive sessions

```typescript
await Flowlines.endSession({
  userId: "user_123",
  sessionId: "sess_456",
});
```

`userId` and `sessionId` must match the values passed to `Flowlines.context()`.

If the app has no concept of sessions that start and end (e.g. a single-shot CLI tool), **do not add `endSession`**. If session boundaries are ambiguous, ask the user.
