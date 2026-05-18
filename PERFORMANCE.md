# PocketSound Bot — Architecture Capabilities

## V1 → V2 → V3

---

## V1: Single Workflow (Monolith)

One n8n workflow. 485 nodes. Everything in one canvas.

**How it worked:**
- Telegram Trigger node received the update
- n8n closed the HTTP connection immediately
- Bot response was sent as a **separate outbound HTTP request** back to Telegram API

```
Telegram ──► n8n Telegram Trigger  (connection closed)
                      │
                      ▼ (separate request, new connection)
n8n ──────────────► Telegram API  (send message)
```

**Two network round trips per every user interaction.**

**Limits:**
- Telegram rate limit: **30 requests/second** outbound
- At 30 active users simultaneously, the bot hits the ceiling
- One broken node could corrupt unrelated features
- No way to test a single feature in isolation
- Database migrations were run directly from the workflow canvas

---

## V2: Modular Architecture (Current)

23 isolated sub-workflows. Webhook node + `Respond to Webhook` pair.

**How it works:**
- Webhook node receives the update and **holds the HTTP connection open**
- Webhook Receiver routes the update to the correct handler
- Handler executes
- `Respond to Webhook (200 OK)` replies **inline** — through the same open connection

```
Telegram ──► Webhook Receiver  ──► Handler
                    │                  │
                    └──────────────────┘
                    Respond to Webhook (200 OK)
                    (same connection, no new request)
```

**One network round trip per user interaction.**

This matches Telegram's official recommendation:
> *"You can use the reply to the webhook request instead of a separate API call."*

---

### What Changes in Numbers

| Metric | V1 | V2 |
|---|---|---|
| Workflows | 1 | 23 |
| Nodes total | 485 | ~519 |
| Largest single workflow | 485 nodes | 79 nodes (`/edit`) |
| Network round trips per interaction | 2 | 1 |
| Outbound API calls consumed per response | 1 (Telegram Trigger) + 1 (send message) | 1 (inline reply) |
| Telegram rate limit ceiling (30 req/s) | ~15 interactions/sec (2 calls each) | ~30 interactions/sec (1 call each) |
| Public webhook endpoints exposed | 1 (Telegram Trigger, always open) | 1 (Webhook Receiver only) |
| Workflows listening to the internet | 1 | 1 |
| Workflows offline (no public endpoint) | 0 | 22 |
| Independent workflow testing | ❌ | ✅ |
| Isolated failures | ❌ | ✅ |

---

### Security Model

In V1 the Telegram Trigger node is an always-on listener — one active public endpoint processing everything.

In V2 only the **Webhook Receiver** is active and publicly reachable. All 22 handler workflows are `active: false`. They have no public endpoint and cannot be triggered from the internet. They execute only when called internally by the Webhook Receiver via `Execute Workflow`.

```
Internet
   │
   ▼
[ Webhook Receiver ]  ← only publicly exposed workflow
   │
   │  internal n8n call only, no public endpoint
   ▼
[ Handler: /start ]   active: false
[ Handler: audio  ]   active: false
[ Handler: /edit  ]   active: false
        ...           active: false  (22 workflows)
```

**Attack surface: 1 endpoint instead of N.**

Any request that doesn't match a known Telegram update structure is dropped at the Switch node in the Webhook Receiver before reaching any handler.

---

### Throughput Model

Telegram imposes a global rate limit of **30 Bot API requests per second**.

In V1, every user interaction costs 2 API calls (receive + respond separately). Effective ceiling: **~15 interactions/second**.

In V2, every user interaction costs 1 API call (inline webhook reply). Additional API calls within a handler (e.g. `copyMessage`, `sendMessage` to admin group) still count, but the response to the user is free.

Effective ceiling for delivering a result to a user: **~30 interactions/second**.

For the `download_track` handler specifically — which sends audio to admin storage, copies to user, and notifies admin — the cost is 3 API calls. But the user-facing response latency is still determined by the inline reply, not a queued outbound call.

---

## V3: Planned Improvements

V3 does not change the webhook architecture — that model is already optimal. V3 focuses on **internal efficiency, security hardening, and maintainability**.

---

**V3:**


### 1. Guard: check_user — Eliminate ~100 Duplicate Nodes

**Current state (V2):**
A 5-node ban/admin check block is copy-pasted into 20 of 23 workflows:

```
Check user in DB  →  User banned?  →  Admin mode active?
      →  Send banned message  →  Notify admin
```

20 workflows × 5 nodes = **100 nodes** doing the same thing.
Changing ban logic (e.g. new message text) requires editing 20 workflows.

**V3:**
Single `Guard: check_user` sub-workflow.

```
Input:  { chat_id }
Output: { allowed: boolean, user: object, reason: "banned" | "admin_mode" | null }
```

Every handler calls the guard at entry. If `allowed = false` the guard handles messaging the user and notifying admin internally, then the handler exits.

| Metric | V2 | V3 |
|---|---|---|
| Nodes implementing ban check | ~100 | ~5 (guard only) |
| Workflows to edit when ban logic changes | 20 | 1 |
| DB queries for ban check per request | 1 per handler | 1 (guard, shared) |

---

### 2. Handler: /edit — Split into 3 Workflows

**Current state (V2):**
One workflow, 79 nodes, handles 5 distinct features:
list playlists for editing, rename playlist, delete playlist, list tracks for deletion, delete track.

**V3 split:**

| Workflow | Nodes (est.) | Responsibility |
|---|---|---|
| `Handler: Edit — List` | ~15 | Show playlists with edit buttons, route to rename or delete |
| `Handler: Edit — Rename` | ~12 | Set rename state, prompt user, handle cancel |
| `Handler: Edit — Delete` | ~20 | Delete playlist or track, confirmation flow |

Largest single workflow drops from **79 → ~25 nodes**.

---

### 3. Helper: register_user — Deduplicate /start

**Current state (V2):**
`Handler: /start` contains two nearly identical registration sequences — one for direct `/start` and one for referral `/start`. ~25 nodes duplicated.

**V3:**
`Helper: register_user` sub-workflow handles the shared logic.

```
Input:  { chat_id, username, first_name, last_name, is_premium, language_code, referred_by? }
Output: { user_id, referral_link, is_new_user }
```

`Handler: /start` calls it once for both paths, passes `referred_by` only when a referral code is present.

---

### 4. Parameterized SQL

**Current state (V2):**
Several workflows build SQL via direct expression interpolation:

```sql
WHERE id = {{ $json.file_id }}
WHERE tg_id = {{ $json.chat_id }}
```

**V3:**
All `WHERE` clauses using user-supplied values use the n8n Postgres node's built-in parameterized query support. Safer and easier to audit.

---

### 5. Remove Double Ban Check in callback_query

**Current state (V2):**
`callback_query` checks ban/admin mode, then calls sub-handlers (`get_file`, `more_results`, etc.) which each check ban/admin mode again.

**2 DB queries per button press** where 1 is enough.

**V3:**
After `Guard: check_user` is introduced, `callback_query` calls the guard once and passes the `user` object to sub-handlers in the payload. Sub-handlers called exclusively from `callback_query` skip the guard — they trust the check was already done.

Sub-handlers that can also be called directly from Webhook Receiver still call the guard themselves.

---

### V3 Summary

| Metric | V2 | V3 (est.) |
|---|---|---|
| Total workflows | 23 | 27 |
| Total nodes | ~519 | ~380 |
| Nodes implementing ban check | ~100 | ~5 |
| Workflows to edit for ban logic change | 20 | 1 |
| Largest single workflow | 79 nodes | ~25 nodes |
| DB queries per button press (ban check) | 2 | 1 |
| Duplicated registration nodes (/start) | ~25 | 0 |
