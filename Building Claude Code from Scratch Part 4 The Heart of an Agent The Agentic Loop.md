

https://mp.weixin.qq.com/s/zEj3WD1oCadrIghosIIH5g



# Building Claude Code from Scratch · Part 4: The Heart of an Agent — The Agentic Loop

_Read this chapter in the novel reader_  
_Go read_  
_Immersive reading mode in the novel reader_

This is Part 4 of the **“Building Claude Code from Scratch”** series, organized and written while studying the open-source project **easy-agent**.

You ask Claude Code: _“Help me refactor this function.”_  
It reads files, edits code, runs tests, finds an error, and edits again.

No one manually controls it in the middle.  
A loop is running automatically.

This article breaks that loop down.

## Why a Loop Is Needed

Part 1 explained that model replies have two kinds of `stop_reason`:

- `end_turn`: finished speaking; this round is done
- `tool_use`: needs to call a tool; not done yet

When it’s `tool_use`, the model is basically saying:  
“I need to read file content first. Please fetch it and send it back, then I’ll continue.”

At this point, you can’t just end the conversation.  
You must execute the tool, append the result to `messages`, and let the model continue.  
The model may ask for another tool again; you execute it again, append again… until `stop_reason` becomes `end_turn`.

This loop — **“execute tool → append result → continue reasoning”** — is the **Agentic Loop**.

## Full Loop Code

The `query` function in easy-agent implements this loop:

```ts
export async function* query({ messages, model, systemPrompt, toolContext, maxTurns = 8 }) {
  const state = {
    messages: [...messages],  // copy so original array is untouched
    turnCount: 0,
  };

  while (state.turnCount < maxTurns) {
    state.turnCount += 1;

    // 1. Send request to model
    const stream = streamMessage({
      messages: state.messages,
      model,
      system: systemPrompt,
      tools: getToolsApiParams(),
    });

    // 2. Receive streaming output and yield it upward for real-time UI rendering
    let result;
    while (true) {
      const { value, done } = await stream.next();
      if (done) { result = value; break; }
      yield value;  // pass through streaming events to UI layer
    }

    // 3. Append assistant reply to history
    state.messages.push(result.assistantMessage);
    yield { type: "assistant_message", message: result.assistantMessage };

    // 4. If not tool_use, assistant is done, exit loop
    if (result.stopReason !== "tool_use") {
      return { state, usage: result.usage, reason: "completed" };
    }

    // 5. Execute tools, append results, continue next turn
    const toolResultMessage = await runTools(result.assistantMessage.content, toolContext);
    state.messages.push(toolResultMessage);
    yield { type: "tool_result_message", message: toolResultMessage };
  }

  // Force exit after max turns
  return { state, usage: { input_tokens: 0, output_tokens: 0 }, reason: "max_turns" };
}
```

Now let’s break it down.

## Step 1: Send Request

```ts
const stream = streamMessage({
  messages: state.messages,
  model,
  system: systemPrompt,
  tools: getToolsApiParams(),  // tell the model which tools are available
});
```

Each round sends the full `messages` history, including all previous tool calls and tool results.  
The model needs this context to know what it has already done and what to do next.

## Step 2: Receive Streaming Output and Pass Through

```ts
while (true) {
  const { value, done } = await stream.next();
  if (done) { result = value; break; }
  yield value;
}
```

A key design point here: `query` itself is also an async generator, and directly `yield`s underlying streaming events.

So the UI can receive text chunks in real time and render as it generates, without waiting for the entire Agentic Loop to finish.  
The loop may run multiple rounds, but users still see progress continuously.

## Step 3: Decide Whether to Continue

```ts
if (result.stopReason !== "tool_use") {
  return { state, usage: result.usage, reason: "completed" };
}
```

`stop_reason` is the control switch for the entire loop.

If it is not `tool_use`, exit immediately — including `end_turn` (normal finish), `max_tokens` (output limit reached), `stop_sequence` (stop sequence triggered), etc.

## Step 4: Execute Tools

```ts
const toolResultMessage = await runTools(result.assistantMessage.content, toolContext);
state.messages.push(toolResultMessage);
```

`runTools` iterates over all `tool_use` blocks in the model reply and executes them one by one:

```ts
export async function runTools(contentBlocks, toolContext) {
  const results = [];

  for (const block of contentBlocks) {
    if (block.type !== "tool_use") continue;

    const tool = findToolByName(block.name);
    if (!tool) {
      results.push({
        type: "tool_result",
        tool_use_id: block.id,  // must match the tool_use id
        content: `Error: unknown tool ${block.name}`,
        is_error: true,
      });
      continue;
    }

    const result = await tool.call(block.input, toolContext);
    results.push({
      type: "tool_result",
      tool_use_id: block.id,
      content: result.content,
      ...(result.isError ? { is_error: true } : {}),
    });
  }

  return { role: "user", content: results };
}
```

`tool_use_id` is critical. Every `tool_result` must include the matching `tool_use` id so the model knows which tool call this result belongs to.  
A single response may contain multiple tool calls, and IDs define the mapping.

Tool results are wrapped as a message with `role: "user"`.  
This is required by the Anthropic API: tool results are sent back as `user`, not `assistant`.

## Why `maxTurns` Exists

```ts
while (state.turnCount < maxTurns) {
```

Without a turn limit, if the model gets stuck in a loop (keeps calling tools, keeps getting errors, keeps retrying), the process may never exit.

`maxTurns = 8` is a safety valve.  
After 8 rounds, force exit and return `reason: "max_turns"` so the upper layer can inform the user.

## Full `messages` Structure

After several rounds, `messages` typically looks like this:

```ts
[
  { role: "user",      content: "Help me refactor this function" },
  { role: "assistant", content: [{ type: "tool_use", id: "tu_1", name: "Read", input: {...} }] },
  { role: "user",      content: [{ type: "tool_result", tool_use_id: "tu_1", content: "..." }] },
  { role: "assistant", content: [{ type: "tool_use", id: "tu_2", name: "Edit", input: {...} }] },
  { role: "user",      content: [{ type: "tool_result", tool_use_id: "tu_2", content: "..." }] },
  { role: "assistant", content: [{ type: "text", text: "Refactoring complete. I made the following changes..." }] },
]
```

Each round’s tool call and tool result stays in history, so the model can always see the full execution trace.

## What This Step Establishes

The Agentic Loop is the core of the whole system.  
The communication layer (Step 1) talks to the model, and the tool layer (Step 3) performs actions.  
The Agentic Loop connects these two layers, enabling the model to autonomously **“think → act → observe → think again.”**

All advanced capabilities later on — permission control, multi-turn orchestration, context compression — are built on top of this loop.

Project: `github.com/ConardLi/easy-agent`

---

Follow me — this series updates daily.

Next article: the complete toolset implementation — **Read, Write, Edit, Grep, Glob, Bash**.  
Among them, **Edit** is the hardest: instead of overwriting the whole file, it must precisely replace a specific segment. If replacement logic is wrong, it can silently corrupt files while the model remains unaware.

(Interface footer text omitted.)
