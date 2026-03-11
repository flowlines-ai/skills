---
name: integrate-flowlines-sdk-python
description: Integrates Flowlines observability / memory SDK into Python LLM applications. Use when adding Flowlines telemetry, instrumenting LLM providers, or setting up OpenTelemetry-based LLM monitoring.
---

# Flowlines SDK for Python — Integration Guide

Flowlines is an observability and memory SDK for LLM-powered Python applications. It instruments LLM provider APIs using OpenTelemetry, automatically capturing requests, responses, timing, and errors, and exports them to the Flowlines backend.

## Step 1: Install the SDK

Requires Python 3.10+.

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

Available extras: `openai`, `anthropic`, `google-generativeai`, `bedrock`, `cohere`, `vertexai`, `together`, `groq`, `mistralai`, `ollama`, `replicate`, `transformers`, `sagemaker`, `watsonx`, `writer`, `alephalpha`, `voyageai`, `openai-agents`, `pinecone`, `chromadb`, `qdrant`, `lancedb`, `marqo`, `milvus`, `weaviate`, `langchain`, `llamaindex`, `crewai`, `agno`, `haystack`, `mcp`.

## Step 2: Initialize the SDK

**CRITICAL: `flowlines.init()` MUST be called BEFORE creating any LLM client** (e.g., `OpenAI()`, `Anthropic()`). If the client is created first, its calls will not be captured.

`flowlines.init()` must be called exactly once. A second call raises `RuntimeError`.

### Mode A — No existing OpenTelemetry setup (default)

Use this when the project does NOT already have its own OpenTelemetry `TracerProvider`. **This is the most common case.**

```python
import flowlines

flowlines.init(api_key="<FLOWLINES_API_KEY>")
```

This auto-detects installed LLM libraries, instruments them, and exports LLM-related spans to Flowlines.

### Mode B — Existing OpenTelemetry setup

Use this **only** when the project already manages its own `TracerProvider`. Pass `has_external_otel=True` to prevent the SDK from creating a second one.

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

### Init parameters

```python
flowlines.init(
    api_key: str,                    # Required. The Flowlines API key.
    ingest_endpoint: str = "https://ingest.flowlines.ai",  # Ingest backend URL.
    api_endpoint: str = "https://api.flowlines.ai",  # API backend URL (memory, sessions).
    has_external_otel: bool = False,  # True if project has its own TracerProvider.
    verbose: bool = False,            # True to enable debug logging to stderr.
)
```

## Step 3: Add context to LLM calls

Wrap LLM calls in `flowlines.context()` to tag spans with user/session/agent IDs:

```python
with flowlines.context(user_id="user-42", session_id="sess-abc"):
    response = client.chat.completions.create(model="gpt-4", messages=messages)
```

`user_id` is required. `session_id` and `agent_id` are optional.

For cases where a context manager doesn't fit (e.g., across request boundaries in web frameworks), use the imperative API:

```python
token = flowlines.set_context(user_id="user-42", session_id="sess-abc")
try:
    client.chat.completions.create(...)
finally:
    flowlines.clear_context(token)
```

**Context does NOT auto-propagate to child threads/tasks.** Set it explicitly in each thread or async task.

### How to find values for user_id, session_id, and agent_id

1. **Look for existing data** in the codebase:
   - `user_id`: the end-user making the request (e.g., authenticated user ID, email, API key owner)
   - `session_id`: the conversation or session grouping multiple interactions (e.g., chat thread ID, conversation UUID)
   - `agent_id`: the AI agent or assistant handling the request (e.g., agent name, assistant ID)

2. **If obvious mappings exist**, use them directly:
   ```python
   with flowlines.context(user_id=request.user.id, session_id=thread_id):
       ...
   ```

3. **If mappings are unclear**, ask the user which variables or fields should be used.

4. **If no data is available yet**, use placeholder values with TODO comments:
   ```python
   with flowlines.context(
       user_id="anonymous",  # TODO: replace with actual user identifier
       session_id=f"sess-{uuid.uuid4().hex[:8]}",  # TODO: replace with actual session/conversation ID
   ):
       ...
   ```

## Step 4: Retrieve and inject memory

Retrieve what Flowlines remembers about a user from previous conversations:

```python
memory = flowlines.get_memory("user-42", session_id="sess-abc")
# or with more context:
memory = flowlines.get_memory("user-42", session_id="sess-abc", agent_id="agent-1", view="summary")
```

For async code:

```python
memory = await flowlines.aget_memory("user-42", session_id="sess-abc")
```

Both return a JSON string of the memory object, or `None` if no memory exists or if an error occurs.

Inject the memory into your prompt so the LLM can personalize its responses:

```python
messages = [{"role": "system", "content": "You are a helpful assistant."}]
if memory:
    messages.append({
        "role": "system",
        "content": f"Here is what you know about this user from previous conversations:\n{memory}",
    })
messages.append({"role": "user", "content": user_input})
```

## Step 5: End the session

When a conversation session is over, signal it to the backend. This flushes pending spans and notifies Flowlines:

```python
flowlines.end_session("user-42", session_id="sess-abc")
```

For async code:

```python
await flowlines.aend_session("user-42", session_id="sess-abc")
```

Look for places in the codebase where a session naturally ends:
- An explicit "end conversation" or "close session" action
- A WebSocket disconnect handler
- A cleanup/logout handler
- A timeout that expires inactive sessions

If the application has no concept of sessions (e.g., a single-shot CLI tool), **skip this step**.

## Full example

```python
import flowlines
from openai import OpenAI

flowlines.init(api_key="your-flowlines-api-key")
client = OpenAI()

user_id = "user-42"
session_id = "sess-abc"

# Retrieve memory for this user
memory = flowlines.get_memory(user_id)

messages = [{"role": "system", "content": "You are a helpful assistant."}]
if memory:
    messages.append({
        "role": "system",
        "content": f"Here is what you know about this user from previous conversations:\n{memory}",
    })
messages.append({"role": "user", "content": "Hello!"})

with flowlines.context(user_id=user_id, session_id=session_id):
    response = client.chat.completions.create(model="gpt-4", messages=messages)

flowlines.end_session(user_id=user_id, session_id=session_id)
```

## Common mistakes to avoid

- Do NOT create the LLM client before calling `flowlines.init()` — spans will be missed.
- Do NOT call `flowlines.init()` more than once — it raises `RuntimeError`.
- Do NOT forget to install the instrumentation extras for the providers you use (e.g., `flowlines[openai]`).
- Do NOT assume context propagates to child threads — set it explicitly in each thread/task.

## Verifying trace ingestion

If the user provides a Flowlines API key, you can verify that traces are being received by the backend:

```bash
curl -X GET 'http://api.flowlines.ai/v1/get-traces' -H 'x-flowlines-api-key: <FLOWLINES_API_KEY>'
```

Use this after the integration is complete and the application has made at least one LLM call, to confirm that traces are flowing correctly.
