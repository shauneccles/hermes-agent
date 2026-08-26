# Spike: Telegram tappable buttons for webhook delivery

**Status:** spike / design proposal — no code changed.
**Branch:** `spike/webhook-telegram-buttons`
**Scope:** let a webhook subscription attach Telegram inline-keyboard buttons to its
delivered message, and route a tap back into the agent.

---

## Summary

Hermes already renders Telegram inline keyboards in four places (exec approvals,
slash confirms, clarify prompts, the model/choice pickers) and has a fifth,
platform-abstract version of the same idea in the relay connector's `prompt` op.
Webhook delivery cannot use any of them, for one structural reason: every existing
button flow resolves a **blocked agent thread** or performs a **local side effect**,
whereas a webhook-delivered message is produced by a run that has already
finished — `WebhookAdapter.on_processing_complete` closes the per-delivery session
the moment the run returns (`gateway/platforms/webhook.py:986`). A tap therefore has
no thread to unblock and no session to rejoin; it has to *start* something. On top of
that, the one seam webhook delivery actually uses — `adapter.send(chat_id, content,
metadata)` at `gateway/platforms/webhook.py:1474` — has no button channel at all:
Telegram's generic `send()` (`plugins/platforms/telegram/adapter.py:5253`–`5609`)
never sets `reply_markup`.

**Feasibility verdict: medium — this needs a small new gateway-side callback router,
not just an adapter tweak.** The button *rendering* is a genuine ~40-line reuse of
existing machinery. The *return path* is new work: a pending-button registry, a `wb:`
callback arm in the Telegram dispatcher, and a re-entry point on the webhook adapter
that synthesises a fresh `MessageEvent` the same way `_handle_webhook` does today
(`gateway/platforms/webhook.py:941`–`974`). Every piece has a close in-tree template
to copy — the relay adapter's `_mint_prompt` / `_pop_prompt` registry
(`gateway/relay/adapter.py:2587`–`2650`) is almost exactly the registry needed — so
the risk is low, but the honest size is roughly **700 lines across 5 files plus
tests**, not a one-file change.

---

## Current state

### 1. The webhook delivery path

**Inbound.** A POST lands on `/webhooks/{route_name}` (or the multiplexed
`/p/{profile}/webhooks/{route_name}`), both registered in `connect()` at
`gateway/platforms/webhook.py:291`–`299`, and is handled by `_handle_webhook`
(`gateway/platforms/webhook.py:634`). In order:

| Step | Location |
|---|---|
| Hot-reload dynamic subscriptions (mtime-gated) | `webhook.py:637` → `_reload_dynamic_routes` `webhook.py:504` |
| Resolve/validate the `/p/<profile>/` prefix | `webhook.py:643` → `_resolve_request_profile` `webhook.py:563` |
| Route lookup, profile binding, `enabled: false` check | `webhook.py:640`, `654`, `671` |
| Body-size cap (header + actual bytes) | `webhook.py:678`–`701` |
| **HMAC validation** (fails closed on missing secret) | `webhook.py:707`–`724` → `_validate_signature` `webhook.py:1078` |
| Rate limit (fixed window, default 30/min) | `webhook.py:728` → `_record_rate_limit_hit` `webhook.py:436` |
| Parse payload (JSON, form-encoded fallback) | `webhook.py:734`–`747` |
| Event-type filter + route filters + optional script | `webhook.py:757`, `769`, `785` |
| Render the prompt template | `webhook.py:810` → `_render_prompt` `webhook.py:1259` |
| Derive `delivery_id` from provider headers | `webhook.py:844`–`850` |
| **Idempotency** (TTL cache, 1h) | `webhook.py:855` → `_record_delivery_id` `webhook.py:451`, TTL at `webhook.py:223` |

**Two terminal modes then diverge:**

*`deliver_only` (no agent).* The rendered prompt **is** the message. It builds a
delivery dict inline and calls `_direct_deliver` (`webhook.py:870`–`920`, helper at
`webhook.py:1317`), returning 200/502 synchronously.

*Agent mode.* This is the path that matters:

1. Session key is minted as `webhook:{route_name}:{delivery_id}` —
   `webhook.py:924`. The `delivery_id` is baked in deliberately so concurrent
   webhooks on one route get independent runs.
2. **Delivery info is stashed for `send()`** — `webhook.py:929`–`938`. This dict
   (`{"deliver": ..., "deliver_extra": ...}`) is the delivery envelope. It is read
   with `.get()`, never popped, so interim status messages don't consume it
   (`webhook.py:360`–`367`).
3. A `SessionSource` + `MessageEvent` are built — `webhook.py:941`–`956`.
4. The run is fire-and-forget: `asyncio.create_task(self.handle_message(event))`,
   and the HTTP call returns `202 Accepted` — `webhook.py:972`–`984`.

**Outbound.** When the agent's final response is ready, the gateway calls
`WebhookAdapter.send(chat_id, content, ...)` (`webhook.py:353`). It:

- drops silence-marker responses (`webhook.py:369`, predicate at `webhook.py:71`),
- looks up the stashed envelope by `chat_id` (`webhook.py:375`),
- dispatches on `deliver_type`: `log`, `github_comment`, or cross-platform
  (`webhook.py:378`–`402`),
- and for Telegram lands in `_deliver_cross_platform` (`webhook.py:1420`), which
  resolves the target adapter, falls back to the home channel if `deliver_extra` has
  no `chat_id` (`webhook.py:1455`–`1466`), and finally calls:

```python
# gateway/platforms/webhook.py:1468-1474
metadata = None
thread_id = extra.get("message_thread_id") or extra.get("thread_id")
if thread_id:
    metadata = {"thread_id": thread_id}
return await adapter.send(chat_id, content, metadata=metadata)
```

**This last line is the only seam.** Both the agent path and the `deliver_only` path
converge on it (`webhook.py:1341`). The `metadata` dict currently carries exactly one
key, `thread_id`, and Telegram's `send()` reads it via `_metadata_thread_id`
(`plugins/platforms/telegram/adapter.py:5325`). Nothing in `send()` between
`adapter.py:5253` and `adapter.py:5609` ever sets `reply_markup` — the first
`reply_markup` in the file is the update prompt at `adapter.py:6281`.

**Session teardown.** `on_processing_complete` → `_end_webhook_session` marks the
session ended with reason `webhook_complete` (`webhook.py:986`–`1072`). The docstring
is explicit that "a webhook delivery is one-shot: the `delivery_id` is baked into the
session key, so the session will never receive a second turn" (`webhook.py:990`).
Any callback design has to reckon with that.

### 2. How clarify/approval buttons work today

There are **two independent halves**: an outbound render method that stores state
keyed by an id, and an inbound arm in `_handle_callback_query` that looks the id up
and resolves a primitive.

**Outbound render.** All four Telegram button senders share one shape:

| Sender | Location | callback_data | State store |
|---|---|---|---|
| `send_exec_approval` | `adapter.py:6307` | `ea:{choice}:{approval_id}` | `self._approval_state[approval_id] = session_key` (`:6376`) |
| `send_slash_confirm` | `adapter.py:6383` | `sc:{choice}:{confirm_id}` | `self._slash_confirm_state[confirm_id]` (`:6425`) |
| `send_clarify` | `adapter.py:6431` | `cl:{clarify_id}:{idx}` / `cl:{clarify_id}:other` | `self._clarify_state[clarify_id]` (`:6507`) |
| `send_choice_picker` | `adapter.py:6587` | `cp:{i}` | `self._choice_picker_state[chat_id]` (`:6639`) |

Each builds `InlineKeyboardButton`/`InlineKeyboardMarkup` (imported at
`adapter.py:144`, lazily rebound at `adapter.py:356`–`376`) and sends through
`_send_message_with_thread_fallback(**kwargs)` (`adapter.py:6215`), a thin
`**kwargs` pass-through to `bot.send_message` that retries once without
`message_thread_id` on "thread not found". Because it forwards arbitrary kwargs, it
already accepts `reply_markup` — it is the natural helper for any new button send.

`send_clarify` is worth reading closely (`adapter.py:6431`–`6511`) because it solved
the two problems a webhook button design will hit:

- **Label truncation.** Full option text goes in the message *body* as a numbered
  list; the buttons carry short numeric labels (`adapter.py:6459`–`6467`, `6480`–`6486`).
- **The 64-byte `callback_data` cap.** Explicitly noted at `adapter.py:6477`; the
  payload is `cl:<id>:<idx>` and the *index* is what travels, with the real choice
  text looked up server-side.

**Inbound dispatch.** `_handle_callback_query` (`adapter.py:7175`) is registered once
as a `CallbackQueryHandler` at `adapter.py:4361` and is a flat prefix chain:

```
mp:/mpg:/mpv:/mm:/mc:/mb/mx/mg:  → model picker      (adapter.py:7191)
cp:                              → choice picker     (adapter.py:7198)
gt:                              → gmail triage      (adapter.py:7205)
ea:                              → exec approval     (adapter.py:7217)
sc:                              → slash confirm     (adapter.py:7302)
cl:                              → clarify           (adapter.py:7402)
update_prompt:                   → updater y/n       (adapter.py:7518)
```

The `wb:` prefix this spec proposes is free.

Every arm follows the same five beats. Using the `cl:` arm (`adapter.py:7402`–`7515`)
as the reference:

1. **Parse** — `data.split(":", 2)` (`:7403`).
2. **Authorize** — `_is_callback_user_authorized(caller_id, chat_id, chat_type,
   thread_id, user_name)` (`:7409`). Defined at `adapter.py:1171`: it reconstructs a
   `SessionSource` and delegates to the runner's `_is_user_authorized`, falling back
   to a `TELEGRAM_ALLOWED_USERS` env allowlist that **fails closed** when unset
   (`adapter.py:1213`–`1221`).
3. **Look up state, one-shot** — `self._clarify_state.get(...)`; a missing entry
   answers "already resolved" (`:7419`–`7422`).
4. **Resolve the primitive** — `resolve_gateway_clarify(clarify_id, resolved_text)`
   (`:7486`), which sets the entry's response and fires the `threading.Event` that
   the agent thread is blocked on (`tools/clarify_gateway.py:164`–`176`).
5. **Ack and strip** — `query.answer(...)` plus `edit_message_text(...,
   reply_markup=None)` so the keyboard can't fire twice (`:7493`–`7499`).

**The blocking primitive.** `tools/clarify_gateway.py` is module-level state
(`_entries`, `_session_index`, `_lock` at `:69`–`73`) so adapters can resolve without
a back-reference to the runner. `register` (`:80`) creates the entry, the agent
thread blocks in `wait_for_response` (`:107`) — which polls in 1-second slices so the
inactivity watchdog keeps getting touched (`:114`–`120`) — and a button callback or
text intercept fires `resolve_gateway_clarify` (`:164`). Timeout default is 3600s
(`:554`–`574`). Wiring lives in `gateway/run.py:5958`–`6026`: `_clarify_callback_sync`
registers the entry, pauses typing, flushes buffered prose, then schedules
`ctx._status_adapter.send_clarify(...)` on the event loop and blocks.

**Reusable vs. clarify-specific:**

| Reusable as-is | Clarify-specific |
|---|---|
| `InlineKeyboardButton`/`Markup` construction + row pairing (`adapter.py:6349`–`6352`) | The `threading.Event` block-and-resolve model (`clarify_gateway.py:107`, `:164`) |
| `_send_message_with_thread_fallback` kwargs pass-through (`adapter.py:6215`) | `_session_index` / text-intercept coercion (`clarify_gateway.py:286`–`463`) |
| The prefix-dispatch + auth + one-shot-pop + ack-and-strip pattern (`adapter.py:7402`–`7515`) | `mark_awaiting_text` "Other" flow (`clarify_gateway.py:466`) |
| Index-in-`callback_data`, text-looked-up-server-side (`adapter.py:6477`) | `_notify_clarify_expired` copy |
| Numbered-list-in-body for long labels (`adapter.py:6459`) | The per-session notify hook (`clarify_gateway.py:585`–`605`) |

Two more precedents matter:

- **`gt:` (gmail triage)** — `adapter.py:7575`. The only existing callback that
  triggers *work* rather than resolving a waiter. It shells out to a script
  (`adapter.py:7615`–`7625`) and distinguishes sticky verbs (keep the keyboard) from
  one-shots (strip it) at `adapter.py:7659`–`7665`. Closest shape to a webhook
  button, but it runs a script, not an agent.
- **The relay `prompt` op** — `docs/relay-connector-contract.md:400`, `:439`–`472`.
  This is the platform-abstract version of everything above:
  `_mint_prompt(kind, state, timeout_s)` → `<owner-nonce>.<8 hex>` id
  (`gateway/relay/adapter.py:2587`), `_pop_prompt` enforcing TTL and one-answer-wins
  (`:2630`), `_minted_here` for cross-instance ownership (`:2620`), a bounded
  `_resolved_prompts` FIFO so a double tap is consumed silently (`:2641`), and
  `_consume_prompt_response` dispatching on `kind` to approval/slash-confirm/clarify
  (`:2875`–`3027`). `MessageEvent.prompt_response` (`gateway/platforms/base.py:2435`)
  is the wire field. **This registry is the direct template for the webhook button
  registry.**

### 3. The gap — why none of this reaches webhook delivery

Three distinct blockers:

**(a) No render channel.** Webhook delivery ends at `adapter.send(chat_id, content,
metadata={"thread_id": ...})` (`webhook.py:1474`). Telegram's `send()` has no
`reply_markup` path (`adapter.py:5253`–`5609`), and buttons only exist on the four
bespoke `send_*` methods, none of which the webhook adapter can reach.

**(b) No waiter to resolve.** Approvals and clarifies work because an agent thread is
parked on a `threading.Event`. A webhook message is delivered *after* the run
completes; `on_processing_complete` then ends the session (`webhook.py:986`). There
is nothing to unblock.

**(c) Clarify can't be borrowed as a shortcut.** The tempting shortcut — "let the
webhook run call `clarify` and get buttons for free" — fails on four counts:

1. `ctx._status_adapter` is `_adapter_for_source(source)` (`gateway/run.py:28989`),
   which for a webhook source resolves to the **WebhookAdapter** via transport
   provenance (`gateway/authz_mixin.py:127`–`152`).
2. `WebhookAdapter` has no `send_clarify` override (verified: no `send_clarify` /
   `send_exec_approval` in `gateway/platforms/webhook.py`), so it gets the base
   numbered-text fallback (`gateway/platforms/base.py:4348`–`4407`).
3. That text goes out via `WebhookAdapter.send()` → `_deliver_cross_platform` →
   `TelegramAdapter.send()` — plain text, no keyboard.
4. Even if the user replies, the reply belongs to the **Telegram** session key, while
   the clarify entry is indexed under `webhook:{route}:{delivery_id}`, so
   `attempt_text_response_for_session` (`clarify_gateway.py:428`) never matches. The
   webhook agent thread then blocks for the full 3600s default
   (`clarify_gateway.py:554`) and dies of timeout.

So: clarify from a webhook run is silently one-way today. Worth fixing on its own
merits (see Open Questions), but it is not the path to webhook buttons.

---

## Proposed design

Opt-in at every layer. A route without the new key behaves byte-identically.

### 3.1 Subscription API

Add an optional `buttons` block to the route schema (config.yaml static routes and
`webhook_subscriptions.json` dynamic ones alike — they merge into one dict at
`webhook.py:553`):

```jsonc
{
  "pr-review": {
    "secret": "…",
    "prompt": "Summarise PR {pull_request.number}: {pull_request.title}",
    "deliver": "telegram",
    "deliver_extra": { "chat_id": "-1001234567890" },

    "buttons": {
      "choices": [
        { "label": "👀 Review",  "value": "review"  },
        { "label": "💬 Comment", "value": "comment" },
        { "label": "🔕 Ignore",  "value": "ignore"  }
      ],
      "callback_prompt": "The user tapped '{button.label}' on PR {pull_request.number}. Do the '{button.value}' action.",
      "ttl_seconds": 3600,
      "per_row": 2
    }
  }
}
```

- `choices` — static, route-owned. `label` is what renders; `value` is what the
  callback prompt receives. Validated at `connect()` alongside the existing
  `deliver_only` check (`webhook.py:277`–`284`): non-empty, ≤ some cap (Telegram
  tolerates ~8 rows comfortably), labels ≤ 64 chars.
- `callback_prompt` — the template for the follow-up agent run, rendered by the
  **existing** `_render_prompt` (`webhook.py:1259`) against the *original* payload,
  with two extra tokens `{button.label}` / `{button.value}` injected the same way
  `{event_type}` is at `webhook.py:1287`. Required whenever `choices` is set.
- `ttl_seconds` — how long the buttons stay live. Default to the existing
  `_idempotency_ttl` (3600, `webhook.py:223`) so the two windows agree.
- `per_row` — cosmetic; mirrors the 2-per-row pairing at `adapter.py:6349`–`6352`.

**CLI.** Add `--buttons` (repeatable `label=value`) and `--callback-prompt` to
`wh_sub` in `hermes_cli/subcommands/webhook.py:24`–`64`, persisted by `_cmd_subscribe`
(`hermes_cli/webhook.py:174`–`201`) and shown by `_cmd_list` (`:227`). Note the
existing precedent that `toolsets` is *deliberately not* CLI-exposed so an
agent-created subscription cannot self-grant elevated tools
(`webhook.py:466`–`494`): `buttons` is safe to expose because a callback run inherits
exactly the same toolset resolution as the original route.

### 3.2 Response-to-buttons contract

**Primary: static, route-declared (recommended).** Buttons come from `buttons.choices`
on the subscription. The agent's response is not parsed at all. This is the right
default because:

- it works for `deliver_only` routes (`webhook.py:870`), where there is no agent
  response — and a zero-LLM push notification with tappable actions is arguably the
  highest-value case here;
- it is deterministic, so the callback contract is fixed at subscribe time and can be
  validated at `connect()`;
- it sidesteps the false-positive problem the `MEDIA:` marker convention has already
  paid for once (`gateway/run.py:1815`–`1820` had to make the matcher
  extension-anchored because bare `MEDIA:` tokens in prose were being treated as
  deliverables; `gateway/run.py:1845`–`1849` documents the two layered guards).

**Secondary (defer to a follow-up): agent-emitted.** If a route sets
`buttons.from_response: true`, parse a trailing fenced block off the final response:

````
```hermes-buttons
[{"label": "👀 Review", "value": "review"}]
```
````

A fenced block with a named language is far safer than a bare `BUTTONS:` line —
learn from `MEDIA:`. Stripping would hook the same place `MEDIA:` is stripped before
display (`gateway/stream_consumer.py:1423`–`1436`). **Recommendation: ship static
first.** Dynamic buttons make the callback contract unpredictable and mean an
unvalidated model output decides what a tap can trigger.

### 3.3 The render seam

Extend the delivery envelope, then teach Telegram's `send()` to read it.

**Webhook side** — `_deliver_cross_platform` (`webhook.py:1468`–`1474`):

```python
metadata = {}
thread_id = extra.get("message_thread_id") or extra.get("thread_id")
if thread_id:
    metadata["thread_id"] = thread_id

buttons = delivery.get("buttons")            # from the route config
if buttons and adapter.supports_inline_buttons:
    set_id = webhook_buttons.mint(
        route=..., delivery_id=..., chat_id=chat_id, thread_id=thread_id,
        choices=buttons["choices"], ttl=buttons.get("ttl_seconds", 3600),
    )
    metadata["buttons"] = {"set_id": set_id, "choices": buttons["choices"]}

return await adapter.send(chat_id, content, metadata=metadata or None)
```

`buttons` gets added to the delivery dicts built at `webhook.py:871`–`877`
(`deliver_only`) and `webhook.py:929`–`934` (agent mode).

**Telegram side** — in `send()` (`adapter.py:5253`), build the keyboard from
`metadata["buttons"]` and pass `reply_markup` down to the send call. Two constraints
inherited from `send_clarify`:

- `callback_data` is `wb:{set_id}:{idx}` — the index travels, the label/value are
  looked up in the registry (the 64-byte lesson, `adapter.py:6477`).
- Attach the keyboard to the **last chunk only**. `send()` splits long messages
  (`adapter.py:5310`–`5322`); a keyboard on every chunk would let one tap fire from
  three different messages.

`supports_inline_buttons` is a new class attribute on `BasePlatformAdapter`, default
`False`, `True` on Telegram. Adapters that don't support it get the numbered-text
degradation described in §5.

### 3.4 Callback flow

**New module: `gateway/webhook_buttons.py`** — module-level registry, same shape as
`tools/clarify_gateway.py` (module state so adapters need no runner back-reference)
and the same semantics as the relay registry (`gateway/relay/adapter.py:2587`–`2650`):

```python
mint(route, delivery_id, chat_id, thread_id, choices, payload, ttl) -> set_id
pop(set_id) -> state | None       # one-shot; TTL-expired entries miss
was_resolved(set_id) -> bool      # bounded FIFO, so a double tap is silent
invalidate_route(route) -> int    # called when a subscription is removed/disabled
```

`set_id` is `<per-process nonce>.<8 hex>`, matching the relay id format
(`gateway/relay/adapter.py:2597`–`2599`) and staying inside the 64-byte budget. The
random, server-side-registered id is what makes a forged `callback_data`
unusable — see §4.

**New arm in `_handle_callback_query`** (`adapter.py:7175`), inserted before the
`update_prompt:` arm at `:7518`:

```
wb:{set_id}:{idx}
  1. authorize      → _is_callback_user_authorized(...)          (adapter.py:1171)
  2. chat binding   → query.message.chat_id == state["chat_id"]  (and thread_id)
  3. one-shot pop   → webhook_buttons.pop(set_id); if resolved-before, answer silently
  4. ack + strip    → query.answer(label); edit_message_text(reply_markup=None)
  5. re-enter       → runner.adapters[Platform.WEBHOOK].handle_button_callback(...)
```

Steps 1, 3 and 4 are copied verbatim in spirit from the `cl:` arm
(`adapter.py:7409`, `:7419`, `:7493`–`7499`); step 4's strip-on-one-shot is the
gmail-triage pattern (`adapter.py:7659`–`7665`). Step 2 is new and is the security
addition (§4).

**New method on `WebhookAdapter`** — `handle_button_callback(state, idx, actor)`:

```python
route_config = self._routes.get(state["route"])          # re-check: route may be gone
if not route_config or route_config.get("enabled", True) is False:
    return  # webhook.py:671 semantics, applied at callback time

if not self._record_callback_rate_limit(state["route"], time.time()):
    return  # separate budget — see §4

choice   = state["choices"][idx]
prompt   = self._render_prompt(
    route_config["buttons"]["callback_prompt"],
    {**state["payload"], "button": {"label": choice["label"], "value": choice["value"]}},
    state["event_type"], state["route"],
)

cb_session = f"webhook:{state['route']}:{state['delivery_id']}:cb:{state['set_id']}"
self._delivery_info[cb_session] = {                       # same envelope as webhook.py:929
    "deliver": route_config.get("deliver", "log"),
    "deliver_extra": {"chat_id": state["chat_id"], "thread_id": state["thread_id"]},
}
source = self.build_source(chat_id=cb_session, chat_name=f"webhook/{state['route']}",
                           chat_type="webhook", user_id=f"webhook:{state['route']}",
                           user_name=state["route"])
source.profile = state["profile"]                         # webhook.py:948
event  = MessageEvent(text=prompt, message_type=MessageType.TEXT,
                      source=source, raw_message=state["payload"],
                      message_id=f"{state['delivery_id']}:cb")
task = asyncio.create_task(self.handle_message(event))    # webhook.py:972
```

This is deliberately the *same* code path as an inbound POST
(`webhook.py:924`–`974`): a fresh one-shot session, the delivery envelope stashed
under it so the answer routes back to the same Telegram chat/thread, and
`on_processing_complete` (`webhook.py:986`) reaping it afterwards for free.

**Why a new run and not a continuation.** The original session is closed
(`webhook.py:986`, reason `webhook_complete`) and its key is `delivery_id`-scoped by
design (`webhook.py:990`). A distinct `:cb:` suffix keeps taps concurrent-safe (two
users tapping two buttons on two notifications don't queue behind each other) and
keeps the ghost-session accounting that `_end_webhook_session` fixed
(`webhook.py:996`–`998`) intact. The cost is no conversational context from the
original run — the `callback_prompt` template must carry what the follow-up needs.
Threading in prior context is listed as an open question.

### 3.5 Concrete example

```bash
hermes webhook subscribe pr-review \
  --deliver telegram --deliver-chat-id -1001234567890 \
  --events pull_request \
  --prompt "Summarise PR {pull_request.number}: {pull_request.title}" \
  --buttons "👀 Review=review" --buttons "💬 Comment=comment" --buttons "🔕 Ignore=ignore" \
  --callback-prompt "User tapped '{button.label}' on PR {pull_request.number}. Do the '{button.value}' action."
```

1. GitHub POSTs `pull_request` with `X-Hub-Signature-256`. HMAC validates
   (`webhook.py:1120`–`1125`), rate limit and idempotency pass
   (`webhook.py:728`, `855`).
2. Agent runs on session `webhook:pr-review:8f2a…`, returns a summary.
3. `send()` → `_deliver_cross_platform` mints `set_id = a1b2c3.4d5e6f7a`, registers
   `{route, delivery_id, chat_id, thread_id, choices, payload, profile, expires_at}`,
   and calls `TelegramAdapter.send(..., metadata={"thread_id": …, "buttons": {…}})`.
4. Telegram renders the summary with `[👀 Review] [💬 Comment]` / `[🔕 Ignore]`,
   `callback_data` = `wb:a1b2c3.4d5e6f7a:0|1|2` (28 bytes — well inside the 64-byte
   cap).
5. User taps **Review**. `_handle_callback_query` matches `wb:`, authorizes the
   tapper, confirms `chat_id` matches, pops the set (one-shot), answers
   "👀 Review", strips the keyboard.
6. `WebhookAdapter.handle_button_callback` renders
   `"User tapped '👀 Review' on PR 4821. Do the 'review' action."`, opens session
   `webhook:pr-review:8f2a…:cb:a1b2c3.4d5e6f7a`, stashes the envelope, and starts the run.
7. The follow-up response flows out through the ordinary `send()` path
   (`webhook.py:353`) into the same Telegram chat/thread. Session closes via
   `on_processing_complete`.

### 3.6 Alternative considered — "Option B: blocking clarify"

Make `WebhookAdapter.send_clarify` delegate to the delivery-target adapter and bind
the clarify's `session_key` to the webhook session, so a webhook-run agent calling
`clarify` gets real Telegram buttons and the tap unblocks its thread.

*Smaller* (it reuses `tools/clarify_gateway.py` wholesale and needs no new registry
or router) but **worse for this use case**:

- It pins an agent thread and the running-agent guard for up to `clarify_timeout`
  (3600s default, `clarify_gateway.py:554`), on a lane whose whole premise is that
  nobody is waiting — see the autonomous-silence rationale at `webhook.py:71`–`97`.
- It can't serve `deliver_only` routes (`webhook.py:870`), which have no agent.
- The buttons arrive mid-run, so the notification and its choices can't be one message.

**Recommendation:** ship Option A. Option B is still worth doing separately as a bug
fix, because clarify-from-a-webhook-run is currently a silent 3600s hang (§2.3c).

---

## Security & idempotency

**HMAC does not — and need not — carry into the callback.** The inbound HMAC
(`webhook.py:707`–`724`) authenticates the *event*. A tap arrives over Telegram's
authenticated bot connection, not over HTTP, so there is no new unauthenticated
surface to sign. What matters is that a button set is only ever minted *after* an
HMAC-validated delivery, which the design guarantees: `mint()` is called from
`_deliver_cross_platform`, downstream of `webhook.py:717`. Signing `set_id` with the
route secret would be redundant — the id is 128-bit-random and only meaningful
against a server-side registry, so a forged `callback_data` misses and is answered
"expired".

**Caller authorization.** Reuse `_is_callback_user_authorized` (`adapter.py:1171`)
unchanged. It resolves through the runner's `_is_user_authorized` and, on the env
fallback, **fails closed** when `TELEGRAM_ALLOWED_USERS` is unset
(`adapter.py:1213`–`1221`, fixing #24457). Buttons on a *group* notification are
exactly the shared-picker hazard the choice-picker gate calls out
(`adapter.py:6659`–`6661`) — a webhook button that triggers an agent run is at least
as sensitive as flipping a model, so the same gate applies.

**Chat/thread binding (new).** The registry stores the `chat_id` and `thread_id` the
message was delivered to; the callback arm rejects the tap unless
`query.message.chat_id` matches (and, in a forum, `message_thread_id`). Existing arms
don't do this because their state is session-keyed; a webhook button set is
chat-keyed, and without the check a forwarded/copied keyboard could drive a run
targeting a chat the tapper isn't in.

**Idempotency — the sharp edge.** A double tap must not run the agent twice. Three
layers, all lifted from the relay registry:

1. `pop()` is one-shot: the second tap finds nothing (`relay/adapter.py:2630`–`2639`).
2. `was_resolved()` — a bounded FIFO of recently resolved ids
   (`relay/adapter.py:2641`–`2650`) so the second tap answers "already actioned"
   instead of the misleading "expired".
3. The keyboard is stripped on the first tap (`adapter.py:7665` pattern), so the
   race window is a genuine double-tap, not a lingering button.

Note the asymmetry with the inbound path: `_record_delivery_id` (`webhook.py:451`)
keys on the provider's delivery id and protects against *provider retries*. Callback
idempotency keys on `set_id` and protects against *user double-taps*. Both should use
the same 3600s TTL (`webhook.py:223`) so a button never outlives the window in which
its originating delivery would be deduplicated.

**Rate limiting.** A callback-triggered run **bypasses**
`_record_rate_limit_hit` (`webhook.py:728`) because there is no POST. Add a separate
per-route callback budget — reuse the same fixed-window helper with its own
`_callback_rate_counts` dict and a lower default (say 10/min): the natural ceiling is
already low (one tap per delivered message, keyboard stripped after) but a
compromised or spammy group shouldn't be able to fan out agent runs.

**Route lifecycle.** Re-check the route at callback time, not just at mint time: a
subscription can be removed (`hermes_cli/webhook.py:253`) or disabled
(`webhook.py:671`) between send and tap, and dynamic routes hot-reload on every
request (`webhook.py:504`–`561`). `invalidate_route()` should be called from the
reload path when a route disappears. Secrets live in a 0600 file
(`hermes_cli/webhook.py:51`–`80`) — button state adds no new secret material.

**Profile multiplexing.** The mint must capture `source.profile`
(`webhook.py:948`) and the callback run must restore it, or a tap on a
`/p/<profile>/` route's notification would execute under the default profile's
config, skills and credentials — precisely the confusion `_resolve_request_profile`
fails closed against (`webhook.py:585`–`594`).

---

## Rollout & risks

**Backward compatibility.** Fully opt-in. A route with no `buttons` key produces
`metadata=None` at `webhook.py:1474` exactly as today; Telegram's `send()` takes an
unchanged path. No existing `callback_data` prefix collides with `wb:` (full list at
`adapter.py:7191`–`7518`). No change to the HTTP contract, the subscriptions file
schema (additive key), or session keying for non-button routes.

**Degradation on non-Telegram targets.** `deliver` accepts ~18 platforms
(`webhook.py:104`–`109`) plus plugin-registered ones (`webhook.py:389`–`392`). Guard
on `supports_inline_buttons` and, when false, append a numbered text list to the
message body — the same fallback shape `BasePlatformAdapter.send_clarify` already
uses (`base.py:4386`–`4407`). Those taps can't come back, so either omit the
callback affordance or (better) log once per route that buttons were degraded.
Discord and Slack have real component APIs and are the obvious phase 2.

**What could break:**

| Risk | Mitigation |
|---|---|
| Keyboard on every chunk of a split message → one tap, three sources | Attach `reply_markup` to the final chunk only (`adapter.py:5310`–`5322`) |
| Silence-marker responses (`webhook.py:369`) mint orphan button state | Mint *inside* the send path, after the silence check, never in `_handle_webhook` |
| Button state lost on gateway restart → taps answer "expired" | Accept for v1 (matches relay, `relay/adapter.py:2596`); persist later if painful |
| `callback_data` overflow | `wb:` + 15-char id + index ≈ 28 bytes; assert < 64 at mint, as `send_clarify` documents (`adapter.py:6477`) |
| Long labels truncated in the button strip | Put full text in the body, short labels on buttons (`adapter.py:6459`–`6467`) |
| A tap silently doing nothing (route deleted, TTL expired) | Always `query.answer()` with a reason — never leave a dead tap unacknowledged (`adapter.py:7421`) |

**Testing plan.** Mirror the existing pairs — `tests/gateway/test_telegram_clarify_buttons.py`
and `test_telegram_approval_buttons.py` — which build a `TelegramAdapter` with an
`AsyncMock` bot (`test_telegram_clarify_buttons.py:30`–`35`) and drive
`_handle_callback_query` with a mocked `Update`/`CallbackQuery`
(`:133`–`149`). New files:

1. `tests/gateway/test_webhook_buttons_render.py` — a route with `buttons` reaches
   `TelegramAdapter.send` with `reply_markup` set and a registered `set_id`; a route
   without it sends `metadata=None`; keyboard on last chunk only; non-button adapter
   degrades to a numbered list.
2. `tests/gateway/test_telegram_webhook_button_callback.py` — `wb:` dispatch;
   unauthorized tapper rejected (copy `:167`–`210`); wrong-chat tap rejected;
   double-tap consumed silently; expired set answers cleanly.
3. `tests/gateway/test_webhook_button_reentry.py` — a tap builds the right
   `MessageEvent`/session key, stashes the envelope so the reply routes back, honours
   `enabled: false` and the callback rate limit, and restores `source.profile`.

Sits alongside the existing webhook suite (`test_webhook_adapter.py`,
`test_webhook_deliver_only.py`, `test_webhook_dynamic_routes.py`,
`test_webhook_session_close.py`, `test_webhook_signature_rate_limit.py`).

**Rough size.** Telegram render seam ~40 lines · `gateway/webhook_buttons.py` ~120 ·
`wb:` callback arm ~70 · `handle_button_callback` + schema validation ~120 · CLI ~40 ·
tests ~300. Call it **~700 lines across 5 files**, most of it transcribed from
existing patterns. One focused week, plus review.

---

## Open questions

1. **Restart durability.** Button sets are process-local (as the relay's are,
   `relay/adapter.py:2596`). A gateway restart makes every outstanding notification's
   buttons dead. Given `webhook_subscriptions.json` already persists per-route state
   (`hermes_cli/webhook.py:51`), is a small on-disk button registry worth it, or is
   "expired, please re-trigger" acceptable for v1?
2. **Context on the callback run.** Should the follow-up inherit the original run's
   transcript (a real continuation) rather than starting clean from
   `callback_prompt`? Continuation would need the closed session
   (`webhook.py:986`) to be re-openable, which cuts against the one-shot design.
3. **Group semantics.** In a shared chat, is the first authorized tap authoritative,
   or should a route be able to pin `deliver_extra.button_user_id`? Currently any
   authorized user can tap.
4. **TTL alignment.** Should `buttons.ttl_seconds` be independently configurable, or
   hard-tied to `_idempotency_ttl` (`webhook.py:223`) so the two windows can't drift?
5. **Dynamic buttons.** Ship `from_response` at all, or keep the callback contract
   static-only permanently? (§3.2 recommends static-first; the question is whether
   dynamic ever lands.)
6. **Clarify-from-webhook.** Should the one-way clarify hang (§2.3c) be fixed in this
   workstream or filed separately? It is a real defect independent of buttons.
7. **Cross-platform parity.** Discord components and Slack Block Kit are the obvious
   next targets. Should `metadata["buttons"]` be defined now as the generic
   cross-adapter contract — effectively an in-process sibling of the relay `prompt`
   op (`docs/relay-connector-contract.md:439`) — or kept Telegram-shaped until a
   second platform actually needs it?
8. **`deliver_only` + buttons.** A zero-LLM push whose *button* spends LLM tokens is
   a deliberate asymmetry. Should such routes require an explicit
   `buttons.allow_agent_callback: true` so the cost is opted into at subscribe time?
