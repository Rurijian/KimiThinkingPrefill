# Kimi Thinking Prefill

Client-side SillyTavern extension implementing the
[kimi-k3-jb patch](https://rentry.org/kimi-k3-jb) (`reasoning_content` thinking prefill for
Kimi/Moonshot models) without modifying any server files.

## What it does

Hooks `CHAT_COMPLETION_SETTINGS_READY` and rewrites the outgoing request payload:

1. **Injection** — when the prompt does not end on an assistant message, appends:

   ```json
   { "role": "assistant", "content": "", "reasoning_content": "<your prefill>", "partial": true }
   ```

   The model then *continues thinking* from the prefilled reasoning, per the Moonshot API's
   documented partial-prefill behavior.

2. **Preset parity (patch transform)** — if the prompt already ends on an assistant message
   whose content starts with `<think>`, the think block is moved into `reasoning_content`
   and the message is flagged `partial: true` — the exact transform the server patch adds to
   `addAssistantPrefix`. This means preset-style prefills like
   `<think>I should continue the story.` keep working unchanged.

3. **Preserved thinking (optional)** — re-attaches each prior assistant message's stored reasoning
   (`chat[i].extra.reasoning`) as `reasoning_content` in the outgoing payload, so Kimi's
   [preserved-thinking behavior](https://platform.kimi.ai/docs/guide/use-thinking-models) works in
   multi-turn chats. Off by default; requires the SillyTavern **Show thoughts** toggle to be on
   (reasoning is only persisted then). Note: prior reasoning is billed as input tokens.

### How prior reasoning is matched to messages

The payload carries no message identifiers, so a stored reasoning has to be paired with a message in
the request. Two sources are used, in order of trust:

1. **Core-attached `reasoning`** — newer SillyTavern versions copy the stored reasoning onto the
   payload message built from that same chat entry, so it is aligned by construction. It only needs
   renaming to `reasoning_content`; nothing is guessed. The original `reasoning` field is left in
   place for sources that read that name.
2. **Text matching** — for messages the core left bare (older versions, or reasoning withheld because
   the turn came from a different model/API), chat and payload are walked from the **newest** end.
   A payload message is paired with the newest chat message at or below the cursor with matching
   text; the cursor never advances on a miss, so a message that exists only in the payload (preset
   prefill, injected turn) simply gets nothing and cannot shift the pairing of the others.

Matching from the start of the conversation instead — positional 1:1 pairing — is the bug this
guards against: once SillyTavern drops older messages for context limits, the payload no longer
starts at the same turn as the chat, every later pair is off by the number of trimmed messages, and
old reasoning lands on new turns. Other cases the walk accounts for:

- **Swipe** — the core drops the last chat message from the payload, so it is skipped here too.
- **Hidden turns** (`Symbol.for('ignore')`) never reach the payload and are skipped.
- **Macros** in stored text are substituted before matching, so `{{char}} waves.` matches the
  substituted form the payload carries.
- **Duplicate texts** pair newest-to-newest; if the payload holds more copies than the chat, the
  extras are left bare (a lost attachment, never a swapped one).
- Turns with no stored reasoning are not candidates at all, so a stretch of non-reasoning turns does
  not consume the matching positions of the others.

## Guards (mirroring the patch)

- Only runs when the current model id matches the configurable filter (default `kimi,moonshot`,
  covers `moonshotai/kimi-k3` on OpenRouter and `kimi-k3`/`kimi-k2-*` on the direct Moonshot API).
- Skipped when JSON schema / structured output is active.
- Skipped when tools/function calling are in play.
- Injection only on normal/regenerate/swipe generations (never quiet prompts or impersonation);
  the `<think>` transform additionally applies on Continue.

## Settings

Extensions menu → **Kimi Thinking Prefill**:

- **Enable thinking prefill** — toggle for the prefill features (transform + injection).
  Independent of the re-attach toggle below; either works without the other.
- **reasoning_content prefill** — the thinking text to prefill (plain text, no `<think>` tag needed).
- **Model filter** — comma-separated substrings matched against the model id.
- **Force thinking on for prefilled requests** (default: on) — sets `include_reasoning` on requests
  this extension modifies. **Required**: with thinking disabled the model continues the seeded
  `reasoning_content` with reply text and never reasons (reply shows up inside the reasoning panel).
- **Send all prior assistant reasoning back to the API** (default: off) — the preserved-thinking
  feature above. Reasons are matched to payload messages from the end of the conversation; see
  [How prior reasoning is matched to messages](#how-prior-reasoning-is-matched-to-messages).
- **Log decisions to browser console** — debug output for each guarded decision.

## Verification

Enable debug logging, generate, and check the browser console for
`[KimiThinkingPrefill] Injected reasoning_content prefill: ...`. On the server side (ST terminal
with request logging), the final message should look like:

```json
{ "role": "assistant", "content": "", "reasoning_content": "I should continue the story.", "partial": true }
```

## Notes

- Works with the direct Moonshot source, OpenRouter (Moonshot provider), and Custom endpoints —
  anywhere the model id matches the filter and the API honors `partial`/`reasoning_content`.
- The server patch is *not* required; do not run both (double transforms are harmless but pointless).
