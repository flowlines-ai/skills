---
name: integrate-flowlines-sdk-python
description: Integrates Flowlines observability SDK into Python LLM applications. Use when adding Flowlines telemetry, instrumenting LLM providers, or setting up OpenTelemetry-based LLM monitoring.
---

# Flowlines SDK for Python — Agent Skill

## What is Flowlines

Flowlines is an observability SDK for LLM-powered Python applications. It instruments LLM provider APIs using OpenTelemetry, automatically capturing requests, responses, timing, and errors. It filters telemetry to only LLM-related spans and exports them via OTLP/HTTP to the Flowlines backend.

Supported LLM providers: OpenAI, Anthropic, Google Generative AI (Gemini), AWS Bedrock, Cohere, Vertex AI, Together AI.
Supported frameworks/tools: LangChain, LlamaIndex, MCP, Pinecone, ChromaDB, Qdrant.

## Installation

Requires Python 3.11+.

```bash
pip install flowlines
```

Then install instrumentation extras for the providers used in the project:

```bash
# Single provider
pip install flowlines[openai]

# Multiple providers
pip install flowlines[openai,anthropic]

# All supported providers
pip install flowlines[all]
```

Available extras: `openai`, `anthropic`, `google-generativeai`, `bedrock`, `cohere`, `vertexai`, `together`, `pinecone`, `chromadb`, `qdrant`, `langchain`, `llamaindex`, `mcp`.

## Integration

There are two integration modes. Pick the one that matches the project's OpenTelemetry situation.

### Mode A — No existing OpenTelemetry setup (default)

Use this when the project does NOT already have its own OpenTelemetry `TracerProvider`. This is the most common case.

```python
import flowlines

flowlines.init(api_key="<FLOWLINES_API_KEY>")
```

This single call:
1. Creates an OpenTelemetry `TracerProvider`
2. Auto-detects which LLM libraries are installed and instruments them
3. Filters spans to only export LLM-related telemetry
4. Sends data to the Flowlines backend via OTLP/HTTP

### Mode B1 — Existing OpenTelemetry setup (`has_external_otel=True`)

Use this when the project already manages its own `TracerProvider`. Pass `has_external_otel=True` to prevent the SDK from creating a second one.

```python
import flowlines
from opentelemetry.sdk.trace import TracerProvider

flowlines.init(api_key="<FLOWLINES_API_KEY>", has_external_otel=True)

provider = TracerProvider()

# Add the Flowlines span processor to the existing provider
processor = flowlines.create_span_processor()
provider.add_span_processor(processor)

# Instrument providers using the Flowlines instrumentor registry
for instrumentor in flowlines.get_instrumentors():
    instrumentor.instrument(tracer_provider=provider)
```

- `create_span_processor()` returns a span processor that filters and exports LLM spans to Flowlines. Call it exactly once.
- `get_instrumentors()` returns instrumentor instances for every supported provider library that is currently installed. You can also skip this and register instrumentors yourself.

## Critical rules

1. **Initialize Flowlines BEFORE creating LLM clients.** `flowlines.init()` must run before any LLM provider client is instantiated (e.g., `OpenAI()`, `Anthropic()`). If the client is created first, its calls will not be captured.

2. **Initialize once.** `flowlines.init()` must be called exactly once. A second call raises `RuntimeError`. Do NOT call it multiple times.

3. **`user_id` is mandatory in `context()`.** The context manager requires `user_id` as a keyword argument. `session_id` and `agent_id` are optional.

4. **Context does not auto-propagate to child threads/tasks.** If using threads or async tasks, set context in each thread/task explicitly.

## User, session, and agent tracking

Tag LLM calls with user/session/agent IDs using the context manager:

```python
with flowlines.context(user_id="user-42", session_id="sess-abc", agent_id="agent-1"):
    client.chat.completions.create(...)  # this span gets user_id, session_id, and agent_id
```

`session_id` and `agent_id` are optional:

```python
with flowlines.context(user_id="user-42"):
    client.chat.completions.create(...)
```

For cases where a context manager doesn't fit (e.g., across request boundaries in web frameworks), use the imperative API:

```python
token = flowlines.set_context(user_id="user-42", session_id="sess-abc", agent_id="agent-1")
try:
    client.chat.completions.create(...)
finally:
    flowlines.clear_context(token)
```

`set_context()` / `clear_context()` are module-level functions.

## Context integration guidance

When integrating `flowlines.context()`, you MUST wrap LLM calls with context. Follow these steps:

1. **Identify existing data** in the codebase that maps to `user_id`, `session_id`, and `agent_id`:
   - `user_id`: the end-user making the request (e.g., authenticated user ID, email, API key owner)
   - `session_id`: the conversation or session grouping multiple interactions (e.g., chat thread ID, session token, conversation UUID)
   - `agent_id`: the AI agent or assistant handling the request (e.g., agent name, bot identifier, assistant ID)

2. **If obvious mappings exist**, use them directly. For example, if the app has `request.user.id` and a `thread_id`, wire them in:
   ```python
   with flowlines.context(user_id=request.user.id, session_id=thread_id):
       ...
   ```

3. **If mappings are unclear**, ask the user which variables or fields should be used for `user_id`, `session_id`, and `agent_id`.

4. **If no data is available yet**, propose using placeholder values with TODO comments so the integration is functional and easy to complete later:
   ```python
   with flowlines.context(
       user_id="anonymous",  # TODO: replace with actual user identifier
       session_id=f"sess-{uuid.uuid4().hex[:8]}",  # TODO: replace with actual session/conversation ID
       agent_id="my-agent",  # TODO: replace with actual agent identifier
   ):
       ...
   ```
   Only include fields that are relevant. `session_id` and `agent_id` can be omitted entirely if not applicable.

## Memory retrieval

Retrieve user memory from the Flowlines backend with `get_memory()` (sync) or `aget_memory()` (async):

```python
memory = flowlines.get_memory("user-42")
memory = flowlines.get_memory("user-42", session_id="sess-abc", agent_id="agent-1", view="summary")
```

```python
memory = await flowlines.aget_memory("user-42")
```

Both return a JSON string of the memory object, or `None` if no memory was found (HTTP 404). On other errors, they raise `FlowlinesMemoryError`.

## end_session integration guidance

When integrating Flowlines, look for places in the codebase where a session or conversation naturally ends, and add an `end_session` call there. Follow these steps:

1. **Identify session boundaries** — look for code paths where a user session, conversation, or chat thread is considered finished. Common examples:
   - A chat/conversation endpoint that receives an explicit "end conversation" or "close session" action from the user
   - A WebSocket `on_disconnect` or connection-close handler
   - A cleanup/logout handler where the user's session is torn down
   - A background job or timeout that expires inactive sessions
   - An explicit "done" or "goodbye" flow in a conversational agent loop

2. **If clear session boundaries exist**, add `end_session` (or `aend_session` for async code) at those points. Make sure the `user_id` and `session_id` match what was passed to `flowlines.context()` earlier:
   ```python
   # Sync
   flowlines.end_session(user_id=user_id, session_id=session_id)

   # Async
   await flowlines.aend_session(user_id=user_id, session_id=session_id)
   ```

3. **If session boundaries are ambiguous** (e.g., the app is a simple script or a single-shot CLI tool with no ongoing sessions), **do not add `end_session`**. It is only useful when the application has a concept of sessions that start and end.

4. **If you're unsure** whether a particular code path represents a session boundary, ask the user.

## Initialization parameters

```python
flowlines.init(
    api_key: str,                    # Required. The Flowlines API key.
    ingest_endpoint: str = "https://ingest.flowlines.ai",  # Ingest backend URL.
    api_endpoint: str = "https://api.flowlines.ai",  # API backend URL (memory, sessions).
    has_external_otel: bool = False,  # True if project has its own TracerProvider.
    verbose: bool = False,            # True to enable debug logging to stderr.
)
```

Both endpoints must use HTTPS, unless they target `localhost` / `127.0.0.1` / `::1` (useful for local development).

## Public API summary

| Function | Description |
|-|-|
| `flowlines.init(api_key, ...)` | Initializes the SDK. Must be called once before any LLM calls. |
| `flowlines.context(user_id=..., session_id=..., agent_id=...)` | Context manager to tag spans with user/session/agent. |
| `flowlines.set_context(user_id=..., session_id=..., agent_id=...)` | Imperative context setting; returns a token. |
| `flowlines.clear_context(token)` | Restores previous context using the token. |
| `flowlines.create_span_processor()` | Returns a `SpanProcessor`. Mode B1 only. Call once. |
| `flowlines.get_instrumentors()` | Returns list of available instrumentor instances. |
| `flowlines.get_memory(user_id, ...)` | Retrieves user memory (sync). Returns JSON string or `None`. |
| `flowlines.aget_memory(user_id, ...)` | Retrieves user memory (async). Returns JSON string or `None`. |
| `flowlines.end_session(user_id, ...)` | Signals session end and flushes spans (sync). |
| `flowlines.aend_session(user_id, ...)` | Signals session end and flushes spans (async). |

## Imports

The public API is accessed via the top-level module:

```python
import flowlines
```

## Verbose / debug mode

Pass `verbose=True` to print debug information to stderr:

```python
flowlines.init(api_key="...", verbose=True)
```

This logs instrumentor discovery, span filtering, and export results.

## Verifying trace ingestion

If the user provides a Flowlines API key, you can verify that traces are being received by the backend:

```bash
curl -X GET 'http://api.flowlines.ai/v1/get-traces' -H 'x-flowlines-api-key: <FLOWLINES_API_KEY>'
```

Use this after the integration is complete and the application has made at least one LLM call, to confirm that traces are flowing correctly.

## Common mistakes to avoid

- Do NOT create the LLM client before initializing Flowlines — spans will be missed.
- Do NOT call `flowlines.init()` more than once — it raises `RuntimeError`.
- Do NOT forget to install the instrumentation extras for the providers you use (e.g., `flowlines[openai]`).
- Do NOT assume context propagates to child threads — set it explicitly in each thread/task.
