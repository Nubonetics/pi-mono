# Pi Agent Framework — Implementation-Level Specification

## Purpose

This document is a precise, implementation-level specification of the **Pi Agent Framework** (`pi-mono`), a TypeScript-based multi-provider AI agent framework by Mario Zechner. It is written at a level of detail sufficient for a code generator — given a YAML agent description — to produce source code that is **behaviorally identical** to the framework's implementation.

The framework is organized as a monorepo with three core packages:

| Package | npm Name | Role |
|---|---|---|
| `packages/ai` | `@mariozechner/pi-ai` | Unified multi-provider LLM API layer |
| `packages/agent` | `@mariozechner/pi-agent-core` | Stateful agent with tool execution and event streaming |
| `packages/coding-agent` | `@mariozechner/pi-coding-agent` | Full-featured interactive coding agent (CLI, sessions, extensions, TUI) |

---

## 1. Entry Point & Invocation Lifecycle

### 1.1 Public API Surface

The framework exposes two levels of API:

**Low-level: `Agent` class** (`packages/agent/src/agent.ts`)

```typescript
class Agent {
  prompt(message: AgentMessage | AgentMessage[]): Promise<void>;
  prompt(input: string, images?: ImageContent[]): Promise<void>;
  continue(): Promise<void>;
  steer(m: AgentMessage): void;
  followUp(m: AgentMessage): void;
  abort(): void;
  waitForIdle(): Promise<void>;
  subscribe(fn: (e: AgentEvent) => void): () => void;
  // State mutators
  setSystemPrompt(v: string): void;
  setModel(m: Model<any>): void;
  setThinkingLevel(l: ThinkingLevel): void;
  setTools(t: AgentTool<any>[]): void;
  replaceMessages(ms: AgentMessage[]): void;
  appendMessage(m: AgentMessage): void;
  clearMessages(): void;
  reset(): void;
}
```

**High-level: `AgentSession` class** (`packages/coding-agent/src/core/agent-session.ts`)

```typescript
class AgentSession {
  prompt(text: string, options?: PromptOptions): Promise<void>;
  steer(text: string, images?: ImageContent[]): Promise<void>;
  followUp(text: string, images?: ImageContent[]): Promise<void>;
  sendCustomMessage<T>(message, options?): Promise<void>;
  sendUserMessage(content, options?): Promise<void>;
  compact(customInstructions?: string): Promise<CompactionResult>;
  executeBash(command, onChunk?, options?): Promise<BashResult>;
  setModel(model: Model<any>): Promise<void>;
  cycleModel(direction?): Promise<ModelCycleResult | undefined>;
  setThinkingLevel(level: ThinkingLevel): void;
  newSession(options?): Promise<boolean>;
  switchSession(sessionPath: string): Promise<boolean>;
  subscribe(listener: AgentSessionEventListener): () => void;
  abort(): Promise<void>;
  reload(): Promise<void>;
}
```

### 1.2 Agent Construction

**File:** `packages/agent/src/agent.ts:119-130`

```typescript
constructor(opts: AgentOptions = {}) {
    this._state = { ...this._state, ...opts.initialState };
    this.convertToLlm = opts.convertToLlm || defaultConvertToLlm;
    this.transformContext = opts.transformContext;
    this.steeringMode = opts.steeringMode || "one-at-a-time";
    this.followUpMode = opts.followUpMode || "one-at-a-time";
    this.streamFn = opts.streamFn || streamSimple;
    this._sessionId = opts.sessionId;
    this.getApiKey = opts.getApiKey;
    this._thinkingBudgets = opts.thinkingBudgets;
    this._maxRetryDelayMs = opts.maxRetryDelayMs;
}
```

**Default `AgentState`** (`packages/agent/src/agent.ts:91-101`):

```typescript
{
    systemPrompt: "",
    model: getModel("google", "gemini-2.5-flash-lite-preview-06-17"),
    thinkingLevel: "off",
    tools: [],
    messages: [],
    isStreaming: false,
    streamMessage: null,
    pendingToolCalls: new Set<string>(),
    error: undefined,
}
```

### 1.3 Invocation Context Construction

When `Agent.prompt()` is called (`packages/agent/src/agent.ts:314-345`):

1. **Guard:** If `_state.isStreaming === true`, throw `"Agent is already processing a prompt."`.
2. **Guard:** If no model configured, throw `"No model configured"`.
3. **Normalize input:**
   - If `input` is `Array<AgentMessage>`: use directly as `msgs`.
   - If `input` is `string`: construct `UserMessage` with `content: [{ type: "text", text: input }, ...images]` and `timestamp: Date.now()`.
   - If `input` is single `AgentMessage`: wrap in array.
4. Call `this._runLoop(msgs)`.

### 1.4 The `_runLoop` Method (Agent ↔ Loop Bridge)

**File:** `packages/agent/src/agent.ts:383-529`

This method bridges the `Agent` class to the pure `agentLoop`/`agentLoopContinue` generators.

**Algorithm:**

1. **Create a running promise** (`this.runningPrompt`) so `waitForIdle()` can await it.
2. **Create abort controller:** `this.abortController = new AbortController()`.
3. **Set streaming state:** `isStreaming = true`, `streamMessage = null`, `error = undefined`.
4. **Compute reasoning:** `this._state.thinkingLevel === "off" ? undefined : thinkingLevel`.
5. **Build `AgentContext`:**
   ```typescript
   { systemPrompt, messages: this._state.messages.slice(), tools: this._state.tools }
   ```
6. **Build `AgentLoopConfig`** with model, reasoning, sessionId, thinkingBudgets, maxRetryDelayMs, convertToLlm, transformContext, getApiKey, and two callback closures:
   - `getSteeringMessages`: calls `this.dequeueSteeringMessages()` (respects `skipInitialSteeringPoll` flag).
   - `getFollowUpMessages`: calls `this.dequeueFollowUpMessages()`.
7. **Start the generator:**
   - If `messages` provided: `agentLoop(messages, context, config, signal, streamFn)`.
   - Else: `agentLoopContinue(context, config, signal, streamFn)`.
8. **Consume `for await` event stream**, updating internal state per event type:
   - `message_start`: set `streamMessage`.
   - `message_update`: update `streamMessage`.
   - `message_end`: clear `streamMessage`, call `appendMessage(event.message)`.
   - `tool_execution_start`: add toolCallId to `pendingToolCalls` set.
   - `tool_execution_end`: remove toolCallId from `pendingToolCalls` set.
   - `turn_end`: check for `errorMessage` on assistant messages, set `_state.error`.
   - `agent_end`: set `isStreaming = false`, clear `streamMessage`.
   - For **every** event: call `this.emit(event)` to notify all listeners.
9. **Handle partial messages:** If the loop ended with a non-empty partial message, append it (unless all content blocks are empty, in which case check for abort).
10. **Error handling:** On any exception, create a synthetic error `AssistantMessage` with `stopReason: "aborted" | "error"`, append it, set `_state.error`, and emit `agent_end`.
11. **Finally:** Reset `isStreaming`, `streamMessage`, `pendingToolCalls`, `abortController`; resolve `runningPrompt`.

### 1.5 The `continue()` Method

**File:** `packages/agent/src/agent.ts:350-376`

Used for retries and resuming queued messages.

**Algorithm:**

1. **Guard:** If streaming, throw.
2. **Guard:** If no messages, throw.
3. If last message is `assistant`:
   - Dequeue steering messages → if any, call `_runLoop(steeringMsgs, { skipInitialSteeringPoll: true })` and return.
   - Dequeue follow-up messages → if any, call `_runLoop(followUpMsgs)` and return.
   - Otherwise throw `"Cannot continue from message role: assistant"`.
4. Else: call `_runLoop(undefined)` (continue from existing context).

### 1.6 Steering and Follow-Up Queue Mechanics

**Steering:** Interrupts the agent mid-run. Delivered after current tool execution finishes; remaining queued tool calls are **skipped**.

**Follow-up:** Processed only after the agent would otherwise stop (no more tool calls, no steering messages).

**Dequeue algorithms** (`packages/agent/src/agent.ts:259-287`):

- **`one-at-a-time` mode (default):** Remove and return only the first item from the queue.
- **`all` mode:** Remove and return all items, clearing the queue.

### 1.7 AgentSession.prompt() Lifecycle

**File:** `packages/coding-agent/src/core/agent-session.ts:654-793`

The high-level `AgentSession.prompt()` wraps `Agent.prompt()` with extension handling, validation, and session persistence.

**Algorithm:**

1. **Extension command check:** If `text.startsWith("/")` and `expandPromptTemplates` is true, try to execute as an extension command. If handled, return immediately.
2. **Extension input event:** Emit `input` event to extensions. Extensions can:
   - `"handled"`: abort prompt entirely.
   - `"transform"`: replace text and/or images.
3. **Expand skills and templates:** Expand `/skill:name` commands and prompt templates.
4. **If streaming:** Queue via `steer()` or `followUp()` based on `streamingBehavior` option.
5. **Flush pending bash messages** (deferred bash results that arrived during a previous streaming turn).
6. **Validate model:** Throw if no model selected.
7. **Validate API key:** Throw if no API key for the model's provider.
8. **Pre-prompt compaction check:** If there's a previous assistant message, check if compaction is needed.
9. **Build messages array:**
   - Create `UserMessage` with content + images.
   - Append any `_pendingNextTurnMessages` (custom messages queued for next turn).
10. **Extension `before_agent_start`:** Extensions can inject custom messages and modify the system prompt.
11. **Call `agent.prompt(messages)`**.
12. **Wait for retry:** `await this.waitForRetry()` (blocks until any auto-retry completes).

---

## 2. The Core Loop (ReAct / Reason-Act)

### 2.1 Outer Loop Structure

**File:** `packages/agent/src/agent-loop.ts:104-198`

The `runLoop()` function implements a nested two-loop architecture:

```
OUTER LOOP (while true):
    INNER LOOP (while hasMoreToolCalls || pendingMessages.length > 0):
        [inject pending messages]
        [LLM call → assistant response]
        [if error/aborted → emit turn_end + agent_end, return]
        [if tool calls → execute tools, check steering]
        [emit turn_end]
        [poll steering messages]

    [poll follow-up messages]
    [if follow-ups → set as pendingMessages, continue outer loop]
    [else → break]

emit agent_end
```

### 2.2 Loop Entry Points

**`agentLoop()`** (`packages/agent/src/agent-loop.ts:28-55`):

1. Create `EventStream<AgentEvent, AgentMessage[]>` with completion predicate `event.type === "agent_end"`.
2. Spawn async IIFE:
   - Copy prompts into `newMessages` and `currentContext.messages`.
   - Emit `agent_start`, `turn_start`.
   - For each prompt: emit `message_start` + `message_end`.
   - Call `runLoop(currentContext, newMessages, config, signal, stream, streamFn)`.
3. Return the stream (consumer iterates via `for await`).

**`agentLoopContinue()`** (`packages/agent/src/agent-loop.ts:65-92`):

1. **Guard:** If no messages, throw `"Cannot continue: no messages in context"`.
2. **Guard:** If last message is `assistant`, throw `"Cannot continue from message role: assistant"`.
3. Spawn async IIFE with `newMessages = []` (no new prompts) and call `runLoop()`.

### 2.3 Per-Step Execution (Inner Loop Iteration)

Each iteration of the inner loop:

1. **Emit `turn_start`** (skipped on the very first turn, since it was already emitted).
2. **Inject pending messages** (steering or follow-up):
   - For each: emit `message_start` + `message_end`, push to `currentContext.messages` and `newMessages`.
   - Clear `pendingMessages`.
3. **Stream assistant response** via `streamAssistantResponse()`.
4. **Check termination:** If `message.stopReason === "error" || "aborted"`:
   - Emit `turn_end` + `agent_end`, call `stream.end()`, return.
5. **Extract tool calls:** `toolCalls = message.content.filter(c => c.type === "toolCall")`.
6. **Set `hasMoreToolCalls`** to `toolCalls.length > 0`.
7. **If tool calls present:** Execute via `executeToolCalls()`.
   - Push all `ToolResultMessage` to `currentContext.messages` and `newMessages`.
8. **Emit `turn_end`** with the assistant message and tool results.
9. **Poll steering:** If `steeringAfterTools` from tool execution, use those. Otherwise call `config.getSteeringMessages()`.

### 2.4 Termination Conditions

The loop terminates when **all** of the following are true:

1. `hasMoreToolCalls === false` (no tool calls in last assistant response).
2. `pendingMessages.length === 0` (no steering messages queued).
3. `(await config.getFollowUpMessages()).length === 0` (no follow-up messages queued).

OR when the assistant message has `stopReason === "error"` or `stopReason === "aborted"`.

There is **no maximum iteration guard** in the core loop itself. The loop runs until the LLM stops producing tool calls and no queued messages remain.

### 2.5 "Is Final Response" Predicate

A response is considered final (loop should stop) when the assistant message contains **zero** `toolCall` content blocks:

```typescript
const toolCalls = message.content.filter((c) => c.type === "toolCall");
hasMoreToolCalls = toolCalls.length > 0;
```

Combined with no pending steering/follow-up messages.

---

## 3. Preprocessing Pipeline (Context Transformation)

### 3.1 Pipeline Overview

Before each LLM call, the context goes through a two-stage transformation pipeline:

```
AgentMessage[]
  → transformContext() [optional, AgentMessage[] → AgentMessage[]]
  → convertToLlm() [AgentMessage[] → Message[]]
  → Context { systemPrompt, messages: Message[], tools: Tool[] }
```

**File:** `packages/agent/src/agent-loop.ts:204-237`

### 3.2 Stage 1: `transformContext` (Optional)

**Signature:** `(messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>`

Applied at the `AgentMessage` level before conversion. Used for:
- Context window management (pruning old messages).
- Injecting context from external sources.
- Session-level context transformations.

Configured via `AgentOptions.transformContext` and passed through `AgentLoopConfig.transformContext`.

### 3.3 Stage 2: `convertToLlm`

**Signature:** `(messages: AgentMessage[]) => Message[] | Promise<Message[]>`

Converts custom `AgentMessage` types to LLM-compatible `Message[]`.

**Default implementation** (`packages/agent/src/agent.ts:30-32`):
```typescript
function defaultConvertToLlm(messages: AgentMessage[]): Message[] {
    return messages.filter((m) =>
        m.role === "user" || m.role === "assistant" || m.role === "toolResult"
    );
}
```

**Coding agent implementation** (`packages/coding-agent/src/core/messages.ts:148-195`):
```typescript
function convertToLlm(messages: AgentMessage[]): Message[] {
    return messages.map((m): Message | undefined => {
        switch (m.role) {
            case "bashExecution":
                if (m.excludeFromContext) return undefined;
                return { role: "user", content: [{ type: "text", text: bashExecutionToText(m) }], timestamp };
            case "custom":
                return { role: "user", content: normalizeContent(m.content), timestamp };
            case "branchSummary":
                return { role: "user", content: [BRANCH_SUMMARY_PREFIX + m.summary + BRANCH_SUMMARY_SUFFIX], timestamp };
            case "compactionSummary":
                return { role: "user", content: [COMPACTION_SUMMARY_PREFIX + m.summary + COMPACTION_SUMMARY_SUFFIX], timestamp };
            case "user":
            case "assistant":
            case "toolResult":
                return m;  // pass through
            default:
                return undefined;  // filter out unknown types
        }
    }).filter(m => m !== undefined);
}
```

### 3.4 System Prompt Construction

**File:** `packages/coding-agent/src/core/system-prompt.ts:35-188`

The `buildSystemPrompt()` function assembles the system prompt from multiple sources:

**Algorithm:**

1. **If custom prompt provided** (`options.customPrompt`):
   - Use custom prompt as base.
   - Append `appendSystemPrompt` if present.
   - Append project context files.
   - Append skills section (if `read` tool is available).
   - Append current date/time and working directory.

2. **Else (default prompt):**
   - Build base prompt: `"You are an expert coding assistant operating inside pi..."`.
   - Build **tools list** from `selectedTools` (default: `["read", "bash", "edit", "write"]`), filtered to known tool descriptions.
   - Build **guidelines** dynamically based on which tools are available:
     - If bash but no grep/find/ls: `"Use bash for file operations"`.
     - If bash + grep/find/ls: `"Prefer grep/find/ls tools over bash"`.
     - If read + edit: `"Use read to examine files before editing"`.
     - Etc.
   - Add Pi documentation paths (README, docs, examples).
   - Append `appendSystemPrompt` if present.
   - Append project context files.
   - Append skills section.
   - Append date/time and working directory.

**Extension system prompt modification** (`agent-session.ts:762-788`):

Before each `agent.prompt()` call, the `AgentSession` emits a `before_agent_start` extension event. Extensions can:
- Inject additional custom messages.
- Return a modified system prompt (or `undefined` to use the base prompt).

### 3.5 Tool Registration

**File:** `packages/coding-agent/src/core/agent-session.ts:1909-1998` (`_buildRuntime`)

**Algorithm:**

1. **Create base tools** via `createAllTools(cwd, options)`:
   - Returns `{ read, bash, edit, write, grep, find, ls }` — each an `AgentTool`.
   - Or uses `baseToolsOverride` if provided.
2. **Build `_baseToolRegistry`:** `Map<string, AgentTool>` from base tools.
3. **Load extensions:** Get extensions from `_resourceLoader.getExtensions()`.
4. **Create `ExtensionRunner`** (if extensions or custom tools exist).
5. **Collect registered tools:**
   - Extension-registered tools (via `pi.registerTool()`).
   - SDK custom tools (via `customTools` option).
   - Wrap all into `AgentTool[]` via `wrapRegisteredTools()`.
6. **Build `toolRegistry`:** Merge base tools + wrapped extension tools.
7. **Determine active tools:**
   - Default active: `["read", "bash", "edit", "write"]` (or `baseToolsOverride` keys).
   - If `includeAllExtensionTools`: add all extension tools.
   - Split into `activeBaseTools` and `activeExtensionTools`.
8. **Wrap with extension callbacks** (if `ExtensionRunner` exists):
   - `wrapToolsWithExtensions()` wraps each tool's `execute()` with `tool_call`/`tool_result` extension events.
9. **Set on agent:** `this.agent.setTools(wrappedActiveTools)`.
10. **Rebuild system prompt** with active tool names.

---

## 4. Contents Construction Algorithm

### 4.1 Session-to-Messages Conversion

**File:** `packages/coding-agent/src/core/session-manager.ts:307-414` (`buildSessionContext`)

The `buildSessionContext()` function converts the session entry tree into an `AgentMessage[]` array suitable for the LLM.

**Algorithm:**

1. **Build index:** Create `Map<string, SessionEntry>` from all entries.
2. **Find leaf:** Use `leafId` if provided, else the last entry.
3. **Walk to root:** From leaf, follow `parentId` links to build the `path[]` (root → leaf order).
4. **Extract settings:** Scan path for `thinking_level_change`, `model_change`, and `compaction` entries.
5. **Build messages with compaction handling:**

   **If compaction exists in path:**
   ```
   a. Emit compaction summary as CompactionSummaryMessage (user message with <summary> tags).
   b. Emit "kept" messages: entries from firstKeptEntryId up to (but not including) the compaction entry.
   c. Emit entries after the compaction entry.
   ```

   **If no compaction:**
   ```
   Emit all entries in path order.
   ```

6. **Entry-to-message conversion** (`appendMessage` helper):
   - `message` → push `entry.message` directly.
   - `custom_message` → convert to `CustomMessage` via `createCustomMessage()`.
   - `branch_summary` → convert to `BranchSummaryMessage` with `<summary>` tags.
   - All other entry types (thinking_level_change, model_change, label, custom, session_info) → skipped.

### 4.2 Compaction Summary Format

```
The conversation history before this point was compacted into the following summary:

<summary>
[compaction summary text]
</summary>
```

### 4.3 Branch Summary Format

```
The following is a summary of a branch that this conversation came back from:

<summary>
[branch summary text]
</summary>
```

### 4.4 Custom Message Handling

Custom messages (from extensions) are converted to `UserMessage` in `convertToLlm()`:
```typescript
{ role: "user", content: normalizedContent, timestamp }
```

### 4.5 Bash Execution Messages

Non-excluded bash executions are converted to `UserMessage`:
```
Ran `[command]`
```[output]```
(command cancelled) // if cancelled
Command exited with code [N] // if non-zero exit
[Output truncated. Full output: path] // if truncated
```

Messages with `excludeFromContext: true` (!! prefix) are filtered out.

### 4.6 Tree-Based Context Segregation

The session uses a **tree structure** where each entry has `id` and `parentId`. Branching creates a new path from a shared ancestor. Only entries on the **path from root to current leaf** are included in context.

**Branch switching:** When switching to a different leaf, the path from root to that leaf is computed, and `buildSessionContext()` produces only messages along that path.

---

## 5. LLM Call Mechanics

### 5.1 The `streamAssistantResponse()` Method

**File:** `packages/agent/src/agent-loop.ts:204-289`

**Algorithm:**

1. **Apply `transformContext`** (if configured): `messages = await config.transformContext(messages, signal)`.
2. **Apply `convertToLlm`:** `llmMessages = await config.convertToLlm(messages)`.
3. **Build `Context`:**
   ```typescript
   { systemPrompt: context.systemPrompt, messages: llmMessages, tools: context.tools }
   ```
4. **Resolve API key:** `config.getApiKey?.(config.model.provider) || config.apiKey`.
5. **Call stream function:**
   ```typescript
   const response = await streamFunction(config.model, llmContext, {
       ...config, apiKey: resolvedApiKey, signal
   });
   ```
6. **Consume streaming events:**
   - `start`: Set `partialMessage`, push to `context.messages`, emit `message_start`.
   - `text_start/delta/end`, `thinking_start/delta/end`, `toolcall_start/delta/end`:
     Update `partialMessage`, update last message in `context.messages`, emit `message_update`.
   - `done`/`error`: Get `finalMessage = await response.result()`, replace in context, emit `message_end`, return.

### 5.2 The Streaming Pipeline

**File:** `packages/ai/src/stream.ts`

```
streamSimple(model, context, options)
  → resolveApiProvider(model.api)
  → provider.streamSimple(model, context, options)
  → returns AssistantMessageEventStream
```

### 5.3 `AssistantMessageEventStream`

**File:** `packages/ai/src/utils/event-stream.ts:68-87`

Extends `EventStream<AssistantMessageEvent, AssistantMessage>`:
- **Completion predicate:** `event.type === "done" || event.type === "error"`.
- **Result extraction:** `done` → `event.message`, `error` → `event.error`.

### 5.4 `EventStream<T, R>` Generic

**File:** `packages/ai/src/utils/event-stream.ts:4-66`

A producer-consumer async queue:
- `push(event)`: delivers to waiting consumer or enqueues.
- `end(result?)`: marks stream as done, resolves all waiters.
- `[Symbol.asyncIterator]()`: yields from queue, waits for new events when empty, returns when done.
- `result()`: returns a `Promise<R>` resolved when a completion event is pushed.

### 5.5 API Provider Registry

**File:** `packages/ai/src/api-registry.ts`

```typescript
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

interface ApiProvider<TApi, TOptions> {
    api: TApi;
    stream: StreamFunction<TApi, TOptions>;
    streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}
```

Providers register via `registerApiProvider()`. The registry maps API identifiers (e.g., `"anthropic-messages"`, `"openai-responses"`, `"google-generative-ai"`) to their stream implementations.

### 5.6 `AssistantMessageEvent` Types

**File:** `packages/ai/src/types.ts:195-207`

```typescript
| { type: "start"; partial: AssistantMessage }
| { type: "text_start"; contentIndex: number; partial: AssistantMessage }
| { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
| { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
| { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
| { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
| { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
| { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
| { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
| { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
| { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
| { type: "error"; reason: "aborted" | "error"; error: AssistantMessage }
```

### 5.7 Error Handling at the LLM Level

Errors during the LLM streaming are handled at two levels:

1. **In `streamAssistantResponse`:** The `"error"` event type produces a `finalMessage` with `stopReason: "error"` and `errorMessage`. This is returned normally and checked by the loop.

2. **In `Agent._runLoop` catch block** (`agent.ts:497-519`): Creates a synthetic error `AssistantMessage`:
   ```typescript
   {
       role: "assistant",
       content: [{ type: "text", text: "" }],
       stopReason: signal.aborted ? "aborted" : "error",
       errorMessage: err?.message || String(err),
       usage: { input: 0, output: 0, ... },
       timestamp: Date.now(),
   }
   ```

### 5.8 Auto-Retry on Transient Errors

**File:** `packages/coding-agent/src/core/agent-session.ts:2027-2118`

The `AgentSession` layer implements automatic retry with exponential backoff for transient errors.

**Retryable error detection** (`_isRetryableError`):
- `stopReason` must be `"error"` with a non-empty `errorMessage`.
- Must NOT be a context overflow error.
- Error message must match regex: `/overloaded|rate.?limit|too many requests|429|500|502|503|504|service.?unavailable|server error|internal error|connection.?error|connection.?refused|other side closed|fetch failed|upstream.?connect|reset before headers|terminated|retry delay/i`.

**Retry algorithm** (`_handleRetryableError`):
1. Increment `_retryAttempt`.
2. Create retry promise on first attempt (for `waitForRetry()`).
3. If `_retryAttempt > settings.maxRetries`: emit `auto_retry_end` failure, reset, return false.
4. Compute delay: `baseDelayMs * 2^(attempt - 1)` (exponential backoff).
5. Emit `auto_retry_start`.
6. Remove error message from agent state (keep in session).
7. Sleep for delay (abortable via `_retryAbortController`).
8. Call `setTimeout(() => this.agent.continue(), 0)` to retry.

---

## 6. Tool Dispatch & Execution

### 6.1 Tool Execution Algorithm

**File:** `packages/agent/src/agent-loop.ts:294-378`

The `executeToolCalls()` function processes tool calls **sequentially** (not in parallel):

```typescript
for (let index = 0; index < toolCalls.length; index++) {
    const toolCall = toolCalls[index];
    const tool = tools?.find((t) => t.name === toolCall.name);
    // ...execute and check steering...
}
```

### 6.2 Per-Tool Execution Lifecycle

For each tool call:

1. **Emit `tool_execution_start`:** `{ type: "tool_execution_start", toolCallId, toolName, args }`.
2. **Find tool:** `tools?.find(t => t.name === toolCall.name)`.
3. **Validate arguments:** `validateToolArguments(tool, toolCall)` using AJV.
4. **Execute:**
   ```typescript
   result = await tool.execute(toolCall.id, validatedArgs, signal, (partialResult) => {
       stream.push({ type: "tool_execution_update", toolCallId, toolName, args, partialResult });
   });
   ```
5. **On error:** Catch exception, create error result:
   ```typescript
   { content: [{ type: "text", text: error.message }], details: {} }
   ```
6. **Emit `tool_execution_end`:** `{ type: "tool_execution_end", toolCallId, toolName, result, isError }`.
7. **Construct `ToolResultMessage`:**
   ```typescript
   {
       role: "toolResult",
       toolCallId: toolCall.id,
       toolName: toolCall.name,
       content: result.content,
       details: result.details,
       isError,
       timestamp: Date.now(),
   }
   ```
8. **Emit `message_start` + `message_end`** for the tool result.

### 6.3 Tool Argument Validation

**File:** `packages/ai/src/utils/validation.ts:53-84`

Uses AJV (JSON Schema validator) with `coerceTypes: true` and `strict: false`:

1. If in browser extension environment or AJV unavailable: skip validation, return raw arguments.
2. Compile tool's TypeBox schema via `ajv.compile(tool.parameters)`.
3. `structuredClone(toolCall.arguments)` for safe mutation.
4. Validate — AJV mutates args in-place for type coercion.
5. On failure: format errors with paths and throw descriptive error.

### 6.4 Tool-Not-Found Handling

If a tool is not found in the registry:

```typescript
if (!tool) throw new Error(`Tool ${toolCall.name} not found`);
```

This enters the catch block, producing an error tool result that is sent back to the LLM.

### 6.5 Steering Interruption During Tool Execution

**File:** `packages/agent/src/agent-loop.ts:364-378`

After **each** tool execution (not just at the end), the loop checks for steering messages:

```typescript
if (getSteeringMessages) {
    const steering = await getSteeringMessages();
    if (steering.length > 0) {
        steeringMessages = steering;
        const remainingCalls = toolCalls.slice(index + 1);
        for (const skipped of remainingCalls) {
            results.push(skipToolCall(skipped, stream));
        }
        break;
    }
}
```

**Skipped tool calls** get a result:
```typescript
{ content: [{ type: "text", text: "Skipped due to queued user message." }], details: {} }
```
with `isError: true`.

### 6.6 Extension Tool Wrapping

**File:** `packages/coding-agent/src/core/extensions/wrapper.ts:38-111`

Each tool is wrapped with extension callbacks:

```
[Extension tool_call event] → can BLOCK execution
         ↓
[Original tool.execute()]
         ↓
[Extension tool_result event] → can MODIFY result (content, details)
```

**`tool_call` interception:**
- If any extension returns `{ block: true, reason: "..." }`, the tool throws an Error with the reason.

**`tool_result` interception:**
- Extensions can return `{ content, details }` to override the result.
- On tool error, `tool_result` is emitted with `isError: true` but the original error is still thrown.

---

## 7. Event System & Session Persistence

### 7.1 Agent Event Schema

**File:** `packages/agent/src/types.ts:179-194`

```typescript
type AgentEvent =
    | { type: "agent_start" }
    | { type: "agent_end"; messages: AgentMessage[] }
    | { type: "turn_start" }
    | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
    | { type: "message_start"; message: AgentMessage }
    | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
    | { type: "message_end"; message: AgentMessage }
    | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
    | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
    | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

### 7.2 AgentSession Event Extensions

**File:** `packages/coding-agent/src/core/agent-session.ts:105-117`

```typescript
type AgentSessionEvent =
    | AgentEvent
    | { type: "auto_compaction_start"; reason: "threshold" | "overflow" }
    | { type: "auto_compaction_end"; result: CompactionResult | undefined; aborted: boolean; willRetry: boolean; errorMessage?: string }
    | { type: "auto_retry_start"; attempt: number; maxAttempts: number; delayMs: number; errorMessage: string }
    | { type: "auto_retry_end"; success: boolean; attempt: number; finalError?: string };
```

### 7.3 Session Entry Types

**File:** `packages/coding-agent/src/core/session-manager.ts:42-145`

All session entries share a base:

```typescript
interface SessionEntryBase {
    type: string;
    id: string;           // 8-char hex UUID prefix, collision-checked
    parentId: string | null;  // tree structure link
    timestamp: string;    // ISO 8601
}
```

**Entry types:**

| Type | Fields | Purpose |
|---|---|---|
| `message` | `message: AgentMessage` | User, assistant, or toolResult messages |
| `thinking_level_change` | `thinkingLevel: string` | Thinking level changes |
| `model_change` | `provider, modelId` | Model switches |
| `compaction` | `summary, firstKeptEntryId, tokensBefore, details?, fromHook?` | Compaction markers |
| `branch_summary` | `fromId, summary, details?, fromHook?` | Branch summaries |
| `custom` | `customType, data?` | Extension-specific data (NOT in LLM context) |
| `custom_message` | `customType, content, details?, display` | Extension messages (IN LLM context) |
| `label` | `targetId, label` | User-defined bookmarks |
| `session_info` | `name?` | Session metadata |

### 7.4 Session File Format

Sessions are stored as **JSONL** (JSON Lines) files. Each line is a JSON object.

**First line:** Session header:
```json
{"type":"session","version":3,"id":"uuid","timestamp":"ISO","cwd":"/path"}
```

**Subsequent lines:** Session entries with `id`, `parentId`, `timestamp`, and type-specific fields.

### 7.5 Persistence Algorithm

**File:** `packages/coding-agent/src/core/session-manager.ts:791-809`

The `_persist()` method implements **lazy flush**:

1. **Wait for first assistant message:** If no assistant message exists yet, mark `flushed = false` and return (don't write to disk).
2. **Flush all entries:** On first persist after an assistant arrives, write ALL accumulated entries at once.
3. **Subsequent entries:** Append individual entries via `appendFileSync()`.

This prevents creating session files for abandoned conversations.

### 7.6 Session Persistence in AgentSession

**File:** `packages/coding-agent/src/core/agent-session.ts:337-374`

On `message_end` events:

- `custom` messages → `sessionManager.appendCustomMessageEntry()`.
- `user`/`assistant`/`toolResult` → `sessionManager.appendMessage()`.
- `bashExecution`, `compactionSummary`, `branchSummary` → persisted elsewhere (by their respective flows).

### 7.7 Session Migrations

**File:** `packages/coding-agent/src/core/session-manager.ts:207-268`

Three versions exist:

- **v1 → v2:** Add `id`/`parentId` tree structure (linear chain). Convert `firstKeptEntryIndex` to `firstKeptEntryId`.
- **v2 → v3:** Rename `hookMessage` role to `custom`.

Migrations run in sequence and mutate entries in place. The file is rewritten after migration.

---

## 8. Compaction System

### 8.1 Overview

Compaction reduces context size by summarizing old messages. It's triggered either automatically (threshold or overflow) or manually.

### 8.2 Trigger Conditions

**File:** `packages/coding-agent/src/core/agent-session.ts:1513-1558`

**Case 1 — Context Overflow:**
- Assistant message indicates context overflow (`isContextOverflow()`).
- Same model as current (prevents cross-model overflow cascades).
- Error is not from before an existing compaction.
- Action: Remove error from agent state → compact → auto-retry.

**Case 2 — Threshold:**
- `shouldCompact(contextTokens, contextWindow, settings)` returns true.
- `stopReason !== "error"`.
- Action: Compact → NO auto-retry.

### 8.3 Compaction Settings

```typescript
interface CompactionSettings {
    enabled: boolean;           // default: true
    reserveTokens: number;      // default: 16384
    keepRecentTokens: number;   // default: 20000
}
```

### 8.4 Compaction Result

```typescript
interface CompactionResult<T = unknown> {
    summary: string;
    firstKeptEntryId: string;   // First entry to keep after compaction
    tokensBefore: number;
    details?: T;                // e.g., CompactionDetails with file lists
}
```

### 8.5 Extension Hook

Before compaction, a `session_before_compact` event is emitted. Extensions can:
- Cancel compaction (`{ cancel: true }`).
- Provide their own compaction result (`{ compaction: CompactionResult }`).

After compaction, a `session_compact` event is emitted with the saved entry.

---

## 9. Agent Transfer Mechanism

### 9.1 Design Note

The Pi framework does **not** implement a multi-agent transfer system like Google ADK's `transfer_to_agent`. Instead, it uses a **single-agent** architecture with:

- **Extensions** for adding capabilities.
- **Session branching** for context isolation.
- **Tool wrapping** for interception and modification.

There is no concept of sub-agents, parent agents, or peer agents within the core framework. The `Agent` class is singular — one agent per session.

### 9.2 Session Branching as Pseudo-Transfer

Session branching provides context isolation similar to agent transfers:

- **Fork:** Creates a new session with a `parentSession` reference. The new session starts fresh but can reference the parent.
- **Switch:** Loads a different session file, replacing all messages.
- **Branch summary:** When returning from a branch, a summary message is injected into the parent context.

---

## 10. Callback / Extension System

### 10.1 Core Agent Callbacks

The core `Agent` class uses a simple event subscription model:

```typescript
agent.subscribe((event: AgentEvent) => void): () => void;
```

There are **no** before/after callbacks at the core level. The Agent emits events and consumers react to them.

### 10.2 Extension Event Types

The extension system (`packages/coding-agent/src/core/extensions/`) provides a rich callback system. Extensions register handlers for these events:

| Event | When it fires | Can intercept? |
|---|---|---|
| `agent_start` | Agent begins processing | No |
| `agent_end` | Agent finishes processing | No |
| `turn_start` | Each turn begins | No |
| `turn_end` | Each turn ends (with message + tool results) | No |
| `tool_call` | Before tool execution | Yes — can block |
| `tool_result` | After tool execution | Yes — can modify result |
| `input` | Before user input is processed | Yes — can handle or transform |
| `before_agent_start` | Before agent.prompt() is called | Yes — can inject messages and modify system prompt |
| `session_start` | Session initialized | No |
| `session_shutdown` | Session shutting down | No |
| `session_before_switch` | Before switching sessions | Yes — can cancel |
| `session_switch` | After session switch | No |
| `session_before_compact` | Before compaction | Yes — can cancel or provide custom compaction |
| `session_compact` | After compaction | No |
| `session_before_fork` | Before session fork | Yes — can cancel |
| `session_before_tree` | Before tree navigation | Yes — can cancel |
| `model_select` | Model changed | No |
| `resources_discover` | Resource discovery phase | Yes — can add paths |

### 10.3 Extension Tool Wrapping Pipeline

For each tool call, the execution order is:

```
1. Extension tool_call handlers (can block)
2. tool.execute() (actual tool)
3. Extension tool_result handlers (can modify result)
```

### 10.4 Extension Commands

Extensions can register slash commands via `pi.registerCommand()`. These are executed immediately (not queued), even during streaming. They have access to an `ExtensionCommandContext` for session control.

---

## 11. Multi-Agent Orchestration

### 11.1 Single-Agent Architecture

Pi uses a **single-agent** design. There is no built-in multi-agent orchestration, sequential pipelines, or agent graphs. A single `Agent` instance manages the entire conversation.

### 11.2 Context Isolation via Session Branching

The session tree structure provides isolation:

- Each entry has `id` and `parentId`, forming a tree.
- **Branching:** Creating a new entry with a `parentId` pointing to a non-leaf creates a branch.
- **Context includes only the root-to-leaf path.** Messages on other branches are excluded.
- **Branch summaries** can be injected when returning from a branch.

### 11.3 State Management for Resumable Invocations

Sessions are fully serializable (JSONL files). To resume:

1. `SessionManager.setSessionFile(path)` loads all entries from the file.
2. `buildSessionContext()` reconstructs `messages`, `thinkingLevel`, and `model` from the entry tree.
3. `Agent.replaceMessages()` sets the reconstructed messages.
4. The agent is ready to accept new prompts.

Model and thinking level are restored from the most recent `model_change` and `thinking_level_change` entries in the path.

---

## Appendix A: Complete Message Type Hierarchy

### A.1 LLM Messages (`@mariozechner/pi-ai`)

```typescript
// packages/ai/src/types.ts

interface UserMessage {
    role: "user";
    content: string | (TextContent | ImageContent)[];
    timestamp: number;
}

interface AssistantMessage {
    role: "assistant";
    content: (TextContent | ThinkingContent | ToolCall)[];
    api: Api;
    provider: Provider;
    model: string;
    usage: Usage;
    stopReason: StopReason;  // "stop" | "length" | "toolUse" | "error" | "aborted"
    errorMessage?: string;
    timestamp: number;
}

interface ToolResultMessage<TDetails = any> {
    role: "toolResult";
    toolCallId: string;
    toolName: string;
    content: (TextContent | ImageContent)[];
    details?: TDetails;
    isError: boolean;
    timestamp: number;
}

type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

### A.2 Custom Messages (`@mariozechner/pi-coding-agent`)

```typescript
// packages/coding-agent/src/core/messages.ts

interface BashExecutionMessage {
    role: "bashExecution";
    command: string;
    output: string;
    exitCode: number | undefined;
    cancelled: boolean;
    truncated: boolean;
    fullOutputPath?: string;
    timestamp: number;
    excludeFromContext?: boolean;
}

interface CustomMessage<T = unknown> {
    role: "custom";
    customType: string;
    content: string | (TextContent | ImageContent)[];
    display: boolean;
    details?: T;
    timestamp: number;
}

interface BranchSummaryMessage {
    role: "branchSummary";
    summary: string;
    fromId: string;
    timestamp: number;
}

interface CompactionSummaryMessage {
    role: "compactionSummary";
    summary: string;
    tokensBefore: number;
    timestamp: number;
}
```

### A.3 Extensible AgentMessage Union

```typescript
// packages/agent/src/types.ts

interface CustomAgentMessages {
    // Empty by default — extended via declaration merging
}

type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

The coding agent extends this via:
```typescript
declare module "@mariozechner/pi-agent-core" {
    interface CustomAgentMessages {
        bashExecution: BashExecutionMessage;
        custom: CustomMessage;
        branchSummary: BranchSummaryMessage;
        compactionSummary: CompactionSummaryMessage;
    }
}
```

---

## Appendix B: Session Entry Schema Reference

### B.1 Session Header

```typescript
interface SessionHeader {
    type: "session";
    version?: number;     // Current: 3
    id: string;           // UUID
    timestamp: string;    // ISO 8601
    cwd: string;
    parentSession?: string;
}
```

### B.2 Session Entry Base

```typescript
interface SessionEntryBase {
    type: string;
    id: string;            // 8-char hex, collision-checked
    parentId: string | null;
    timestamp: string;     // ISO 8601
}
```

### B.3 All Entry Types

```typescript
type SessionEntry =
    | SessionMessageEntry      // type: "message"
    | ThinkingLevelChangeEntry // type: "thinking_level_change"
    | ModelChangeEntry         // type: "model_change"
    | CompactionEntry          // type: "compaction"
    | BranchSummaryEntry       // type: "branch_summary"
    | CustomEntry              // type: "custom" (NOT in LLM context)
    | CustomMessageEntry       // type: "custom_message" (IN LLM context)
    | LabelEntry               // type: "label"
    | SessionInfoEntry;        // type: "session_info"
```

---

## Appendix C: AgentLoopConfig Schema Reference

```typescript
interface AgentLoopConfig extends SimpleStreamOptions {
    model: Model<any>;
    convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;
    transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
    getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined;
    getSteeringMessages?: () => Promise<AgentMessage[]>;
    getFollowUpMessages?: () => Promise<AgentMessage[]>;

    // Inherited from SimpleStreamOptions:
    reasoning?: ThinkingLevel;
    thinkingBudgets?: ThinkingBudgets;

    // Inherited from StreamOptions:
    temperature?: number;
    maxTokens?: number;
    signal?: AbortSignal;
    apiKey?: string;
    cacheRetention?: CacheRetention;
    sessionId?: string;
    onPayload?: (payload: unknown) => void;
    headers?: Record<string, string>;
    maxRetryDelayMs?: number;
}
```

---

## Appendix D: Content Type Schema Reference

```typescript
interface TextContent {
    type: "text";
    text: string;
    textSignature?: string;
}

interface ThinkingContent {
    type: "thinking";
    thinking: string;
    thinkingSignature?: string;
}

interface ImageContent {
    type: "image";
    data: string;      // base64
    mimeType: string;  // e.g., "image/jpeg"
}

interface ToolCall {
    type: "toolCall";
    id: string;
    name: string;
    arguments: Record<string, any>;
    thoughtSignature?: string;  // Google-specific
}

interface Usage {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    totalTokens: number;
    cost: {
        input: number;
        output: number;
        cacheRead: number;
        cacheWrite: number;
        total: number;
    };
}
```

---

## Appendix E: Model Schema Reference

```typescript
interface Model<TApi extends Api> {
    id: string;
    name: string;
    api: TApi;
    provider: Provider;
    baseUrl: string;
    reasoning: boolean;
    input: ("text" | "image")[];
    cost: {
        input: number;       // $/million tokens
        output: number;
        cacheRead: number;
        cacheWrite: number;
    };
    contextWindow: number;
    maxTokens: number;
    headers?: Record<string, string>;
    compat?: OpenAICompletionsCompat | OpenAIResponsesCompat;
}
```

---

## Appendix F: Tool Schema Reference

```typescript
// Base tool (pi-ai)
interface Tool<TParameters extends TSchema = TSchema> {
    name: string;
    description: string;
    parameters: TParameters;  // TypeBox schema
}

// Agent tool (pi-agent-core) — adds execution
interface AgentTool<TParameters extends TSchema, TDetails = any> extends Tool<TParameters> {
    label: string;
    execute: (
        toolCallId: string,
        params: Static<TParameters>,
        signal?: AbortSignal,
        onUpdate?: AgentToolUpdateCallback<TDetails>,
    ) => Promise<AgentToolResult<TDetails>>;
}

interface AgentToolResult<T> {
    content: (TextContent | ImageContent)[];
    details: T;
}

type AgentToolUpdateCallback<T = any> = (partialResult: AgentToolResult<T>) => void;
```

### F.1 Built-in Tools

| Tool | Name | Parameters | Details |
|---|---|---|---|
| Read | `read` | `{ path, offset?, limit? }` | `ReadToolDetails` |
| Write | `write` | `{ path, content }` | – |
| Edit | `edit` | `{ path, old_text, new_text }` | `EditToolDetails` |
| Bash | `bash` | `{ command }` | `BashToolDetails` |
| Grep | `grep` | `{ pattern, path?, ... }` | `GrepToolDetails` |
| Find | `find` | `{ pattern, path? }` | `FindToolDetails` |
| Ls | `ls` | `{ path? }` | `LsToolDetails` |

---

## Appendix G: Supported LLM Providers

| Provider | API | Identifier |
|---|---|---|
| OpenAI | Completions | `openai-completions` |
| OpenAI | Responses | `openai-responses` |
| Azure OpenAI | Responses | `azure-openai-responses` |
| OpenAI Codex | Responses | `openai-codex-responses` |
| Anthropic | Messages | `anthropic-messages` |
| AWS Bedrock | Converse Stream | `bedrock-converse-stream` |
| Google | Generative AI | `google-generative-ai` |
| Google | Gemini CLI | `google-gemini-cli` |
| Google | Vertex AI | `google-vertex` |

Additional providers (GitHub Copilot, xAI, Groq, Cerebras, OpenRouter, Vercel, Mistral, MiniMax, HuggingFace, Ollama, vLLM, LM Studio) register via the same `openai-completions` or `openai-responses` API with custom base URLs.
