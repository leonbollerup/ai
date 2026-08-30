# qwen-3.8-27b-chromix+froggerick.v1

A tool-handling focused chat template for Qwen 3.8, based on `qwen3.8-safe-v2.1-reasoning-compat` with selected tool-compatibility ideas inspired by the Froggeric fixed Qwen templates.

This version deliberately changes **tool handling only**. It does not try to redesign reasoning, system prompts, role handling, vision/video handling, or normal chat behavior.

## Why this exists

Qwen-style tool calling works well when every client sends the same message format. In practice, LM Studio, vLLM, llama.cpp-based servers and OpenAI-compatible agent frameworks may return tool history in slightly different shapes.

The most important example is `function.arguments`:

- Some histories contain a normal object/mapping.
- OpenAI-compatible histories often contain the same arguments as a JSON **string**.

The original safe template already handles this correctly by automatically switching the rendered tool-call history from Qwen's pseudo-XML format to JSON when a JSON-string argument is detected. This fork keeps that behavior and adds several additional protections around tool serialization.

## What changed

### 1. Safer automatic XML / JSON selection

The safe template already uses `tool_call_format="auto"` by default:

- Mapping arguments -> Qwen pseudo-XML tool format.
- JSON-string arguments -> JSON tool format.

That behavior is preserved.

This fork adds another check: if existing tool data cannot be represented safely inside Qwen's pseudo-XML syntax, auto mode switches the whole tool history to JSON instead.

Examples include:

- A tool name containing characters that would break `<function=...>`.
- A parameter name containing characters that would break `<parameter=...>`.
- A string argument containing reserved markers such as `</parameter>`, `</function>` or `</tool_call>`.

**Expected result:** fewer malformed tool histories and fewer parser failures caused by tool content accidentally looking like Qwen's control tags.

### 2. Forced XML remains strict

If `tool_call_format="xml"` is explicitly requested, the template does not silently rewrite unsafe data.

It raises a clear error when XML cannot represent the tool call safely.

This includes:

- JSON-string arguments that cannot be losslessly decoded by the Jinja environment.
- Invalid XML-style tool names or parameter names.
- String values containing reserved Qwen tool-control delimiters.

**Expected result:** explicit XML mode either produces structurally valid tool syntax or fails early with a useful explanation. It should never quietly corrupt arguments.

### 3. Better protection for tool and parameter names

In XML mode, tool and parameter names are validated before being inserted into:

```text
<function=tool_name>
<parameter=parameter_name>
```

Characters that can break the pseudo-XML structure are rejected.

In auto mode, histories containing unsafe names are rendered with the JSON tool format instead.

**Expected result:** unusual or badly formed names cannot create broken tool tags.

### 4. Reserved delimiter protection inside string arguments

Qwen's XML tool format places string values directly between pseudo-XML tags. A value containing something like:

```text
</parameter>
```

could otherwise terminate the parameter block early.

In auto mode, this fork detects these collisions and switches to JSON tool rendering. In forced XML mode it raises an error.

**Expected result:** file contents, generated code, XML snippets, logs or other tool arguments containing Qwen control strings are not mistaken for tool syntax.

### 5. Clearer parallel / multiple tool-call instructions

The tool instructions now explicitly tell the model that multiple or parallel calls must use one complete block per function:

```text
<tool_call>
...
</tool_call>
<tool_call>
...
</tool_call>
```

Calls must not be nested, and conversational text must not be appended after a tool call.

This applies to both XML and JSON tool-call modes.

**Expected result:** more predictable parsing when the model decides to call several tools in one assistant turn.

### 6. Tool-only fallback when the original user query has fallen out of context

The original safe template intentionally raises:

```text
No user query found in messages.
```

when it cannot find a normal user turn.

That remains the behavior for ordinary non-tool conversations.

However, long-running agents can legitimately retain a sequence such as:

```text
assistant tool call
-> tool result
-> assistant tool call
-> tool result
```

while the original user message has already been removed by context truncation.

This fork detects actual tool activity and, only in that situation, uses the most recent wrapped tool-response turn or the final retained message as the history boundary instead of aborting template rendering.

**Expected result:** an active tool loop does not die solely because the initiating user message fell outside the retained context window.

### 7. Consecutive tool responses remain grouped

Consecutive `role="tool"` messages continue to be rendered inside one Qwen user block as separate `<tool_response>` sections.

Example:

```text
<|im_start|>user
<tool_response>
result A
</tool_response>
<tool_response>
result B
</tool_response><|im_end|>
```

This keeps parallel tool-result ordering intact.

**Expected result:** multi-tool histories remain deterministic and preserve the order of returned results.

### 8. Tool response truncation is preserved

`max_tool_response_chars` remains optional and defaults to `0`, meaning no truncation.

When enabled, oversized tool responses are truncated with the original character count recorded.

`max_tool_arg_chars` is also preserved for XML parameter values.

JSON tool arguments are intentionally kept lossless rather than being cut in the middle of a JSON object, because truncating serialized JSON would make the tool call invalid.

**Expected result:** context size can be controlled without silently producing malformed JSON tool calls.

## What was intentionally NOT changed

This release is intentionally narrow.

It does **not** change:

- Reasoning effort or reasoning prompts.
- Thinking preservation behavior.
- `<think>` generation behavior.
- System/developer message policy.
- Unknown role handling.
- Image/video content handling.
- Model identity or personality.
- General chat formatting.

It also does **not** inject synthetic warnings into tool results or attempt to classify whether a tool call succeeded or failed. Tool output is kept as tool output.

## Tool call IDs

This version does not add OpenAI `tool_call_id` values to the Qwen prompt format.

Qwen's chat-template format represents tool calls and tool responses by ordered blocks rather than by serializing OpenAI tool-call IDs into the prompt. IDs should therefore continue to be handled by the API/runtime layer when required.

## Default behavior

The recommended/default setting remains:

```text
tool_call_format = auto
```

In auto mode the template prefers Qwen's XML-style tool syntax when it can be represented safely, and automatically uses JSON when that is the lossless or structurally safe option.

You can still force a format with:

```text
tool_call_format = xml
```

or:

```text
tool_call_format = json
```

Forced modes are stricter by design.

## Testing

The initial template was compiled with Jinja2 and tested against 12 synthetic tool-history cases covering:

- Normal XML mapping arguments.
- Booleans and numeric XML parameters.
- OpenAI-style JSON-string arguments.
- Explicit XML rejection of JSON-string arguments.
- Reserved XML delimiter collisions in string values.
- Forced-XML delimiter rejection.
- Unsafe tool names.
- Multiple/parallel tool calls.
- Tool-only history after the original user query is gone.
- Preservation of the strict missing-user error for non-tool histories.
- Consecutive/parallel tool responses.
- Tool-response truncation.

All 12 tests passed.

These are template-level tests. Runtime testing with the actual inference stack and agent framework is still recommended.

## Credits / sources

This template is based on:

- `qwen3.8-safe-v2.1-reasoning-compat` from the Chromix Qwen 3.8 work: https://www.reddit.com/r/LocalLLaMA/comments/1voha70/fixedimproved_jinja_chat_template_for_qwen_38/

- Froggeric's fixed Qwen chat-template work, used as a reference for tool-loop robustness and clearer multi-tool instructions:
  https://huggingface.co/froggeric/Qwen-Fixed-Chat-Templates

- Qwen's native tool-call conventions and chat-template format.

The goal of this fork is not to replace either project wholesale. It keeps the strict/lossless behavior of the safe template and applies only tool-specific hardening where it improves compatibility without changing unrelated model behavior.

## File

```text
qwen-3.8-27b-chromix+froggerick.v1.jinja
```
