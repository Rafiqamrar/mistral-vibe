# Context Compaction

Compaction keeps a long conversation inside the model's context window. When the history grows too large, Vibe asks a model to summarize it, then continues the session from that summary instead of the full transcript.

Source: `vibe/core/compaction/` (`manager.py`, `context.py`) and `AutoCompactMiddleware` in `vibe/core/middleware.py`.

## When it runs

- **Automatically.** `AutoCompactMiddleware.before_turn` compares `stats.context_tokens` with the active model's `auto_compact_threshold`. If the threshold is positive and reached, it returns `MiddlewareAction.COMPACT`.
- **Threshold resolution.** A model's own threshold wins. Otherwise a model that declares a context window compacts at a fixed ratio of that window. Otherwise the global `auto_compact_threshold` setting applies. A value `<= 0` disables auto-compaction.
- **Warning.** `ContextWarningMiddleware` injects a one-time warning at 50% of the threshold so the user knows compaction is approaching.

## What `CompactionManager.compact()` does

1. **Snapshot.** `select_model_context` returns the part of the history the model currently sees: the system messages plus everything from the latest compaction boundary onward.
2. **Keep user messages.** `collect_prior_user_messages` selects real user messages to carry over verbatim. It walks newest-first within a budget of `COMPACT_USER_MESSAGE_MAX_TOKENS` (20,000). A message that overflows the budget is middle-truncated. System-injected messages and earlier summaries are skipped. User messages preserved by a previous compaction are parsed back out of the old envelope and merged in.
3. **Summarize.** The compaction prompt (`compaction_prompt_id`) is sent to the compaction model (`get_compaction_model()`). The model must answer with `<summary>...</summary>`, which `extract_summary` pulls out.
4. **Append a boundary.** The result is a single injected user message, tagged `context_boundary="compaction"`, containing the preserved user messages and the summary. `context_tokens` is reset to 0 and the session is saved. The next LLM turn recomputes the real size from API usage.

The live message list is only touched in step 4. All summarization runs on a local copy, so a failure never leaves a half-trimmed conversation.

### The envelope

```
You are continuing a trajectory after a context compaction.
...
<previous_user_messages>
<previous_user_message>...</previous_user_message>
</previous_user_messages>

Here is a summary of what has happened so far:

<compaction_summary>
...
</compaction_summary>
```

Reserved tags inside user content are HTML-escaped so a user message cannot forge the envelope.

## Failure handling

Each model call goes through the injected `complete` callable, so usage is always accounted.

| Situation | Behavior |
|---|---|
| Context too long for the summary call | `drop_oldest_round` removes the oldest user round (with its assistant and tool messages) and retries, up to 3 times. |
| Primary call returns tool calls (`tool_call`) or no `<summary>` (`empty_summary`) | Unless `raise_on_compaction_failure` is set, a fallback runs: a fresh compaction system prompt, thinking off, no tools, and the conversation rendered as a plain-text transcript. This deliberately breaks the prompt cache for a more reliable summary. |
| Fallback also fails | A `compaction_failed` telemetry event is sent with the primary's reason. The placeholder summary `(no summary available)` is used. |
| `raise_on_compaction_failure = true` | No fallback. `CompactionFailedError` is raised. |

The primary call is cache-friendly: the summary request is appended to the live conversation's token prefix, so the provider's prompt cache can be reused.

## Helpers worth knowing

- `reorder_for_tool_adjacency` moves messages that landed between a `tool_use` and its `tool_result` (for example a message steered in mid-tool-call) to just after the results. Providers such as Anthropic reject the gap. It is pure and order-stable, so it is safe to run on every outbound request.
- `render_teleport_summary_request` builds a summary request for teleporting a session to Vibe Code Web.

## Configuration

| Setting | Purpose |
|---|---|
| `auto_compact_threshold` | Token count that triggers compaction for models with no threshold or window of their own. |
| `compaction_model` | Optional model for summaries. It is used only if it shares the active provider. |
| `compaction_prompt_id` | Which prompt drives the summary. |
| `raise_on_compaction_failure` | Fail loudly instead of falling back. Defaults to `false`. |
