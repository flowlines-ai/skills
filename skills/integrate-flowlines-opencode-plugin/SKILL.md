---
name: integrate-flowlines-opencode-plugin
description: Integrates the @flowlines/opencode-plugin into an OpenCode project. Use when adding Flowlines observability to an OpenCode setup, configuring the Flowlines plugin for OpenCode, or when the user asks to monitor AI coding sessions with Flowlines.
---

# Flowlines OpenCode Plugin Integration Guide

This document helps AI coding agents integrate the `@flowlines/opencode-plugin` into an OpenCode project.

## What is the Flowlines OpenCode Plugin?

`@flowlines/opencode-plugin` is an [OpenCode](https://opencode.ai) plugin that captures OpenTelemetry spans and exports them to [Flowlines](https://flowlines.ai) for observability and tracing of AI coding sessions. It initializes the Flowlines SDK, sets up an OpenTelemetry `NodeSDK` with a Flowlines span processor, and automatically exports all LLM-related spans to the Flowlines backend. The SDK is automatically shut down on process exit.

## Prerequisites

- An OpenCode project with a `.opencode/opencode.jsonc` configuration file
- A Flowlines API key

## Installation

There is nothing to install manually. OpenCode resolves plugins automatically from npm when listed in the config.

## Configuration

### Step 1: Enable OpenTelemetry and register the plugin

Edit the `.opencode/opencode.jsonc` file. Add `experimental.openTelemetry: true` and add `@flowlines/opencode-plugin` to the `plugin` array:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "experimental": {
    "openTelemetry": true
  },
  "plugin": ["@flowlines/opencode-plugin"]
}
```

If the file already has other settings, merge these fields into the existing config. If there is already a `plugin` array, append `"@flowlines/opencode-plugin"` to it. If `experimental` already exists, add `"openTelemetry": true` to it.

**Important:** The `experimental.openTelemetry` flag MUST be enabled. Without it, the plugin will not work.

### Step 2: Set the Flowlines API key

The plugin reads the API key from the `FLOWLINES_API_KEY` environment variable. Ensure it is set before running OpenCode:

```sh
export FLOWLINES_API_KEY="your-api-key"
```

Alternatively, add it to a `.env` file if the project supports that.

## Programmatic Usage (with @opencode-ai/sdk)

When using OpenCode programmatically via `@opencode-ai/sdk`, you need to pass the `userId` and `sessionId` so that Flowlines can properly identify users and sessions.

Set `username` in the config and use the session ID when sending prompts:

```ts
import { createOpencode } from "@opencode-ai/sdk";

const userId = "your-user-id";
const sessionId = "your-session-id";

const { client, server } = await createOpencode({
  config: {
    username: userId,
    experimental: {
      openTelemetry: true,
    },
  },
});

const result = await client.session.prompt({
  path: { id: sessionId },
  body: {
    parts: [{ type: "text", text: "Hello!" }],
  },
});
```

- **`username`** maps to the `userId` in Flowlines. Set this to identify which user is making requests.
- **`sessionId`** is passed as `path.id` when calling `client.session.prompt()`. This identifies the conversation session in Flowlines.

## Fetch and use memory

Use `Flowlines.getMemory()` to retrieve memory context for a user. Inject the returned string into your prompt (e.g. as part of the message):

```ts
import { Flowlines } from "@flowlines/sdk";

const memory = await Flowlines.getMemory({
  userId: "user_123",
  sessionId: "sess_456",
  agentId: "agent_1", // optional
});

const result = await client.session.prompt({
  path: { id: sessionId },
  body: {
    parts: [{ type: "text", text: `${memory}\n\nHello!` }],
  },
});
```

`getMemory()` returns a JSON string with the memory content, or an empty string if no memory is found.

## End a session

Look for places where a session or conversation naturally ends and call `Flowlines.endSession()` there. This flushes pending spans and notifies the backend. Common examples:

- An explicit "end conversation" or "close session" action
- A session cleanup or logout handler
- A timeout that expires inactive sessions

```ts
import { Flowlines } from "@flowlines/sdk";

await Flowlines.endSession({
  userId: "user_123",
  sessionId: "sess_456",
});
```

`userId` and `sessionId` must match the values used in the config (`username`) and session prompt (`path.id`).

If the app has no concept of sessions that start and end (e.g. interactive CLI usage), **do not add `endSession`**. If session boundaries are ambiguous, ask the user.

## How It Works

The plugin:
1. Initializes the Flowlines SDK with `hasExternalOtel: true` (since OpenCode already manages OpenTelemetry)
2. Creates a `NodeSDK` with `Flowlines.createSpanProcessor()` and `Flowlines.getInstrumentations()`
3. Starts the SDK to begin capturing spans
4. Registers a `beforeExit` handler to gracefully shut down both Flowlines and the OpenTelemetry SDK

All OpenTelemetry spans emitted by OpenCode are captured and exported to the Flowlines backend.

## Critical Integration Rules

1. **`experimental.openTelemetry` must be `true`.** Without this flag, OpenCode does not emit OpenTelemetry spans and the plugin has nothing to capture.

2. **`FLOWLINES_API_KEY` must be set.** The plugin reads the API key from this environment variable. If it is missing or empty, the plugin will fail to initialize and log an error.

3. **Do NOT install `@flowlines/opencode-plugin` as a project dependency.** OpenCode resolves plugins from npm automatically. Just list it in the `plugin` array in the config file.

4. **For programmatic usage, set `username` in the config.** This is how the plugin identifies the user making requests.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No spans in Flowlines dashboard | `experimental.openTelemetry` not enabled | Add `"experimental": { "openTelemetry": true }` to `.opencode/opencode.jsonc` |
| Plugin initialization error in logs | Missing or invalid `FLOWLINES_API_KEY` | Set the `FLOWLINES_API_KEY` environment variable with a valid API key |
| Plugin not loading | Plugin not listed in config | Add `"@flowlines/opencode-plugin"` to the `plugin` array in `.opencode/opencode.jsonc` |

## Verifying trace ingestion

If the user provides a Flowlines API key, you can verify that traces are being received by the backend:

```bash
curl -X GET 'http://api.flowlines.ai/v1/get-traces' -H 'x-flowlines-api-key: <FLOWLINES_API_KEY>'
```

Use this after the integration is complete and the application has made at least one LLM call, to confirm that traces are flowing correctly.
