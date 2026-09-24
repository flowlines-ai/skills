---
name: integrate-flowlines-claude-agent-sdk-js
description: Integrates Flowlines observability into a TypeScript or Node.js application built on the Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`) by sending Claude Code's built-in OpenTelemetry logs and traces to Flowlines. Use whenever a user wants Flowlines, agent monitoring, session analytics, or tracing for a codebase that calls `query()` from the Claude Agent SDK, even if they do not mention OpenTelemetry. Do not use for the plain Anthropic client SDK (`@anthropic-ai/sdk`), for Python agents, or for a developer's own Claude Code CLI sessions.
---

# Flowlines for Claude Agent SDK applications (TypeScript)

For each `query()`, the Claude Agent SDK starts the Claude Code CLI as a child process. The model calls, tools, and subagents all run in that process, and Claude Code has OpenTelemetry built in. This integration passes environment variables to the child process so that Claude Code exports its own logs and traces directly to Flowlines. The application needs no OpenTelemetry package and no Flowlines SDK.

## What Flowlines receives

With the configuration below, each run sends:

- the user prompts;
- the assistant's text responses, from the main conversation and from subagents;
- every tool call with its input, and the output of tools that export it (file reads and shell commands, up to 60 KB each);
- the model, token counts, cost, and latency of each model request;
- the end-user ID and release version that the application attaches.

Claude Code does not export the system prompt, the tool definitions, or extended thinking.

Flowlines groups the data by Agent SDK session ID. A resumed session stays one Flowlines session, and each prompt in it becomes one turn.

## Consent and secrets

Before you change code:

1. Tell the user what leaves the application (the list above). Prompts, answers, and tool data can contain personal data, customer data, source code, or file content. Get explicit consent. A general request to "add monitoring" is not consent to export content. Offer to leave out tool outputs: remove `OTEL_LOG_TOOL_CONTENT`, and Flowlines still receives tool names and inputs.
2. Confirm that the user has a Flowlines namespace API key. If they do not, tell them to create one in the Flowlines app under Settings, API keys (`https://app.flowlines.ai/settings`), for the namespace that should receive the sessions. The key is shown once. The user stores it in the deployment's secret manager as `FLOWLINES_API_KEY`. Never ask for the key in chat, and never write it into source, examples, tests, or commands. Never print an existing value.
3. The request lets you edit and test the repository. It does not let you deploy, or send live data to Flowlines, unless the user asks for that.

## Inspect the target first

Read the repository's instructions and testing conventions. Then find:

- **Every place that starts Claude Code.** This is usually `query()`, sometimes behind a wrapper. Also look for a custom `spawnClaudeCodeProcess` or a remote sandbox, where the process runs somewhere else.
- **How each call builds `options.env`.** In TypeScript, `env` replaces the inherited environment; the SDK does not merge it with `process.env`. When `env` is omitted, the child inherits `process.env`. Some codebases pass a deliberate allow-list (for example `PATH`, `HOME`, `ANTHROPIC_API_KEY`); keep it.
- **The end user of each run.** Find where a stable user ID is available at the call site: an auth context, the request, a job payload.
- **How conversations continue.** Look for `resume`, `continue`, `sessionId`, or streaming input.
- **`settingSources`.** When it is omitted, Claude Code loads the user, project, and local settings files. The `env` block of those files can override the variables that the application passes. Check `.claude/settings.json` and `.claude/settings.local.json` in the repository, and any `~/.claude/settings.json` that the deployment image ships, for `OTEL_*` or `CLAUDE_CODE_ENABLE_TELEMETRY` entries.
- **Existing OpenTelemetry in the host application.** Its `OTEL_*` variables are inherited by the child process too. Do not change the host's own telemetry.
- **A release identifier available at runtime,** such as the package version or a commit SHA variable.
- **The package manager, test runner, type checker, and linter,** and where deployment environment variables and secrets are declared (`.env.example`, Docker, Kubernetes, Terraform, CI).

## Implement

### 1. Add one helper

Put a small module next to the Agent SDK code. Adapt its name, path, and style to the codebase, and read the key the way the project reads its other secrets:

```ts
const DEFAULT_FLOWLINES_API_BASE_URL = "https://api.flowlines.ai";

type Env = Record<string, string | undefined>;

export interface FlowlinesRunContext {
  /** Stable ID of the end user this run acts for. */
  readonly userId?: string;
  /** Release of this application. Flowlines shows it as the session version. */
  readonly release?: string;
}

/**
 * Environment for the Claude Code process that the Agent SDK starts. Claude Code sends its
 * own OpenTelemetry logs and traces to Flowlines. Without FLOWLINES_API_KEY, the base
 * environment is returned unchanged and nothing is exported.
 */
export function withFlowlinesTelemetry(base: Env, context: FlowlinesRunContext = {}): Env {
  const apiKey = process.env.FLOWLINES_API_KEY;
  if (!apiKey) {
    return base;
  }

  const inherited = Object.fromEntries(
    Object.entries(base).filter(
      ([name]) => !name.startsWith("OTEL_") && name !== "FLOWLINES_API_KEY",
    ),
  );
  const resourceAttributes = [
    ["flowlines.user_id", context.userId],
    ["service.version", context.release],
  ]
    .filter((entry): entry is [string, string] => Boolean(entry[1]))
    .map(([name, value]) => `${name}=${encodeURIComponent(value)}`)
    .join(",");

  return {
    ...inherited,
    CLAUDE_CODE_ENABLE_TELEMETRY: "1",
    CLAUDE_CODE_ENHANCED_TELEMETRY_BETA: "1",
    OTEL_LOGS_EXPORTER: "otlp",
    OTEL_TRACES_EXPORTER: "otlp",
    OTEL_METRICS_EXPORTER: "none",
    OTEL_EXPORTER_OTLP_PROTOCOL: "http/protobuf",
    OTEL_EXPORTER_OTLP_ENDPOINT:
      process.env.FLOWLINES_API_BASE_URL ?? DEFAULT_FLOWLINES_API_BASE_URL,
    OTEL_EXPORTER_OTLP_HEADERS: `x-flowlines-api-key=${apiKey}`,
    OTEL_LOGS_EXPORT_INTERVAL: "1000",
    OTEL_TRACES_EXPORT_INTERVAL: "1000",
    OTEL_LOG_USER_PROMPTS: "1",
    OTEL_LOG_ASSISTANT_RESPONSES: "1",
    OTEL_LOG_TOOL_DETAILS: "1",
    OTEL_LOG_TOOL_CONTENT: "1",
    ...(resourceAttributes ? { OTEL_RESOURCE_ATTRIBUTES: resourceAttributes } : {}),
  };
}
```

Why each part matters:

- **It removes inherited `OTEL_*` variables.** The host's own exporter settings would otherwise reach Claude Code. A signal-specific variable such as `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` takes precedence over the generic Flowlines endpoint. The host's collector would then receive the data together with the Flowlines key.
- **It removes `FLOWLINES_API_KEY`.** Claude Code does not need it; the header variable carries the key.
- **The endpoint is the base URL, with no path.** The exporter adds `/v1/logs` and `/v1/traces`.
- **Traces stay on.** They need `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA`, and tool outputs arrive only through trace span events. Flowlines does not use metrics.
- **Export intervals are short.** Each `query()` runs a CLI process that exits when the run ends. The flush at exit has a short time limit, so shorter intervals leave less data waiting for it.
- **The user ID goes in `flowlines.user_id`, spelled exactly like that.** Without it, Flowlines uses the Claude Code credential's identity, and every session from one deployment looks like the same user. Flowlines does not read `enduser.id`. The value is percent-encoded because `OTEL_RESOURCE_ATTRIBUTES` reserves commas, equals signs, and spaces.
- **`service.version` becomes the session version** in Flowlines, which release comparisons use.
- **With no key, nothing changes,** so local development and tests export nothing.

Never set `console` as an exporter: the SDK uses standard output as its message channel, and telemetry there corrupts the stream. Do not set `OTEL_LOG_RAW_API_BODIES`: Flowlines does not read those events, and they repeat the whole conversation in every request.

### 2. Pass the environment at every call site

```ts
for await (const message of query({
  prompt,
  options: {
    ...options,
    env: withFlowlinesTelemetry(options.env ?? process.env, { userId: user.id, release }),
  },
})) {
  // ...
}
```

- **Build on the environment the call already uses.** If the call passes an explicit allow-list, pass that allow-list as the base, not `process.env`.
- **Take the user ID from this run,** at the call, not at module load. A shared process serves many users, and `env` is fixed for the life of each Claude Code process. A streaming-input session keeps one process for all its turns, so start it with the user it serves.
- **With a custom `spawnClaudeCodeProcess` or a remote sandbox,** make sure the environment reaches the process that runs Claude Code, and that this process can open HTTPS connections to the Flowlines endpoint.

### 3. Keep conversations in one session

The Flowlines session is the Agent SDK session ID.

- **Resumed and streamed conversations stay together.** An application that resumes the session (`resume`) or streams input already keeps a conversation in one Flowlines session.
- **A fresh `query()` for each message makes a new session each time.** Tell the user instead of changing it: resuming changes what the model remembers, so it is a product decision.
- **To tie a new conversation to the application's own ID,** the SDK accepts a UUID in `sessionId`.

### 4. Declare the configuration

Add `FLOWLINES_API_KEY` as a secret, and optionally `FLOWLINES_API_BASE_URL`, wherever the project declares its environment: `.env.example`, deployment manifests, CI. Use names and placeholders only.

## Verify

Add tests in the project's test framework. At minimum, cover these cases:

- **Without `FLOWLINES_API_KEY`,** the base environment comes back unchanged.
- **With a key:**
  - the telemetry switches, exporters, content flags, base endpoint, and header are present;
  - inherited `OTEL_*` variables and `FLOWLINES_API_KEY` are removed;
  - other inherited variables such as `PATH` are kept.
- **User IDs:** a user ID containing a comma, an equals sign, and a space is percent-encoded. Without a user ID, `flowlines.user_id` is left out.
- **Call sites:** when `query()` can be mocked, the call site passes the environment built for the current user.

Use a fake key such as `sk-fl-test`, never a real one. Run the project's type checker, linter, and tests.

Do a live check only if the user asks for it and has set the key outside the chat:

1. Run one harmless prompt with a test user ID.
2. Claude Code drops export errors silently. To see them, add `CLAUDE_CODE_OTEL_DIAG_STDERR=1` to the environment for that run and read the SDK's `stderr` callback. This needs Claude Code 2.1.179 or later.
3. In the Flowlines app, or with the Flowlines MCP server's session tools, confirm that the session shows the test user, the prompt, the answer, and the tool calls. Processing can take a minute.

## Hand off

Report:

- the files and dependencies you changed;
- the environment variables each deployment must set;
- where the user ID comes from, and how conversations map to sessions;
- what is exported, and whether tool outputs are included;
- the checks you ran, and whether you verified receipt in Flowlines;
- these known limits:
  - Flowlines lists these sessions under the agent name `claude-code`, and `OTEL_SERVICE_NAME` does not change it, so several agents in one namespace share that name.
  - The system prompt and thinking are not exported.
  - Commands that the agent runs can inherit the export header. For an agent that runs shell commands on untrusted input, restrict its tools or accept that risk. Flowlines removes the exact key from the telemetry it receives, but a command can still read it.
