# Provider: Anthropic

Triggered by introspection (Workflow step 2.5) when the call-site function imports `anthropic` and calls one of:

- `anthropic.messages.create`
- `client.messages.create` (where `client = anthropic.Anthropic(...)`)
- `Anthropic(...).messages.create`

## `{{PROVIDER_ASSERTS}}` substitution

```python
assert os.getenv("ANTHROPIC_API_KEY"), "ANTHROPIC_API_KEY is required for the wired task_fn."
```

## Optional env vars (do NOT assert; document in `# TODO` comments)

- `ANTHROPIC_BASE_URL` — only needed when pointing at a proxy or self-hosted relay.

## Adapter notes

- `anthropic.Anthropic().messages.create(model=..., max_tokens=..., messages=[...])` returns a `Message` object. `content` is a list of blocks (text, tool_use, thinking, etc.); do not assume block 0 is text.
- If the user's function returns the raw `Message`, wrap it with an extractor that joins every text block, for example `"".join(block.text for block in message.content if getattr(block, "type", None) == "text")`, in `task_fn`. Preserve `tool_use` blocks separately when the experiment evaluates tool use.
- `max_tokens` is **required** — unlike OpenAI, Anthropic raises if it's missing. If the user's function omits it, leave their signature alone; the call will fail at runtime and the user can fix.
- Async (`AsyncAnthropic`): wrap with `asyncio.run(...)` inside a sync `task_fn`.

## Common gotchas

- Tool use response format differs from OpenAI — `tool_use` blocks are interleaved with `text` blocks in `.content`. Don't assume `.content[0]` is text; loop and concatenate `text` blocks if needed.
- Anthropic models require `messages` to alternate user/assistant (no two consecutive same-role messages). If the user's function builds messages, trust it; don't second-guess.
