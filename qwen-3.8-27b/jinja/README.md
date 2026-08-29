# Qwen 3.8 safe-v2 Jinja template — vLLM reasoning compatibility

This directory contains a minimally modified version of Chromix's
`qwen3.8-safe-v2` chat template for Qwen 3.8 models:

- `qwen3.8-safe-v2.1-reasoning-compat.jinja`

Upstream template:

- https://pastebin.com/raw/Qyk6kRv8
- Upstream version identifier: `qwen3.8-safe-v2`

## Why this change was needed

The upstream template preserves reasoning from earlier assistant turns by
reading either of these message fields:

1. `reasoning_content`
2. `thinking`

Some vLLM OpenAI-compatible deployments return parsed reasoning in a field
named `reasoning`. A client may faithfully store and replay that response using
the same field name. The upstream template does not read `message.reasoning`,
so that historical reasoning is silently treated as empty when the next prompt
is rendered.

This matters most during multi-turn tool use. An assistant response can contain
reasoning plus one or more tool calls. After the tool results are appended, the
conversation is rendered again for the next model turn. If the reasoning field
is ignored, the model sees the tool call and result but not the reasoning that
led to the call. The rendered history contains an empty thinking block instead.
That can cause repeated probes, poor convergence, changed tool-selection
behavior, or misleading benchmark results.

This was observed in the local agentic test harness. Ordinary single-turn and
non-tool cases were stable, while the tool-oriented cases were the ones whose
behavior changed. The missing alias meant the comparison was not fully
adapter-correct: a score change could have come from discarded reasoning
history rather than from the intrinsic quality of the template.

## Exact modification

The assistant-history extraction now uses this precedence order:

1. `message.reasoning_content`
2. `message.thinking`
3. `message.reasoning`

The added branch is a fallback only:

```jinja2
{%- elif message.reasoning is defined
         and message.reasoning is not none %}
    {%- if message.reasoning is string %}
        {%- set reasoning_content = message.reasoning %}
    {%- else %}
        {%- set reasoning_content = message.reasoning | string %}
    {%- endif %}
```

The template version identifier was changed to:

```jinja2
qwen3.8-safe-v2.1-reasoning-compat
```

## Why the precedence order is important

`reasoning_content` and `thinking` retain their original priority. Existing
clients therefore receive exactly the same behavior as before. The new
`reasoning` alias is consulted only when neither established field is present.

This avoids ambiguity when a message happens to contain multiple reasoning
fields and prevents the compatibility fix from changing established rendering
behavior.

## What was deliberately not changed

The patch does not alter:

- Qwen's visible chat format or special tokens
- reasoning-level handling
- the default thinking mode
- tool definitions or tool-call formatting
- JSON-string argument handling
- developer/system-message handling
- prefix-cache-sensitive text
- generation-prompt construction
- content rendering

The goal is compatibility, not a redesign of the upstream template.

## Validation performed

The modified template was checked with Jinja2 3.1.6.

The following validations passed:

- The complete template parses successfully as Jinja.
- `reasoning_content`, `thinking`, and `reasoning` render the same preserved
  thinking block when each is supplied independently.
- Existing test messages using `reasoning_content` remain byte-for-byte
  identical between the upstream and modified templates.
- Existing test messages using `thinking` remain byte-for-byte identical.
- Messages without historical reasoning remain byte-for-byte identical.

These checks establish that the new behavior is isolated to messages that use
the previously unsupported `reasoning` field.

## Deployment guidance

Deploy this file as the server's chat template using the mechanism supported by
the serving stack. Keep the reasoning parser enabled if parsed reasoning is
required in API responses.

For non-thinking requests, do not send an unsupported
`reasoning_effort="none"` value to this template. The safe-v2 template accepts
`high`/`xhigh`, `medium`, and `low`. To disable thinking, omit
`reasoning_effort` and pass `enable_thinking=false` through the applicable chat
template arguments.

The client must also preserve the assistant reasoning field when it stores and
replays tool-call history. This template can consume all three common aliases,
but it cannot recover reasoning that the client discarded entirely.

## Recommended verification after deployment

Before accepting benchmark scores as final:

1. Send a multi-turn tool request that produces non-empty parsed reasoning.
2. Confirm the assistant message is replayed with that reasoning intact.
3. Confirm the next rendered turn contains the previous thinking text rather
   than an empty thinking block.
4. Rerun the tool-oriented harness cases T13 through T16.
5. If those results change materially, rerun the full benchmark suite and
   replace the provisional score.

## Privacy note

Preserving reasoning means historical reasoning text is sent back to the model
on later turns. Operators should ensure that this is appropriate for their
logging, retention, and privacy requirements. Set the template's preservation
option according to the deployment's policy when historical thinking should
not be retained.
