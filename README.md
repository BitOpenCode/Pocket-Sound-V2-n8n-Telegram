# 🎵 PocketSound Bot

> Telegram bot for music search, downloads, and playlist management — built entirely on n8n.

---

## From Monolith to Modular — V1 → V2

### V1: The Monolith

The original bot was a **single n8n workflow with 485 nodes**.

Everything lived in one canvas — user registration, music search, Deezer integration, playlist CRUD, admin tooling, download queue, database migrations, and support chat routing. All connected, all tangled. The workflow used the built-in **Telegram Trigger node** to receive updates.

This worked, but had hard limits:

- Any edit risked breaking unrelated features
- Debugging meant scrolling through hundreds of nodes
- Database migrations (`Миграция 15`, `16`, `17`...) were run directly from the workflow canvas
- The Telegram Trigger node **cannot respond to a webhook with a reply** — it only receives

The last point is what forced the architectural rethink.

### The Webhook Problem

Telegram sends updates to n8n via webhooks and imposes a rate limit of **30 API requests per second**. Telegram's own documentation recommends responding to a webhook update **inline** — using the HTTP response itself to issue a Bot API method — rather than making a separate API call back.

> *"You can use the reply to the webhook request instead of using a separate API call. [...] This allows faster response and reduces the number of outgoing requests."*
> — Telegram Bot API docs

The n8n **Telegram Trigger node** does not support this. It receives the update and closes the connection, requiring every bot response to be a separate outbound HTTP call. At scale, this means every user interaction costs an extra round trip and counts against the rate limit.

### V2: Modular Architecture

V2 replaces the Telegram Trigger with a **Webhook node** + `Respond to Webhook` node pair. This allows n8n to hold the HTTP connection open and reply inline, matching Telegram's recommended pattern.

The monolith was decomposed into **23 isolated sub-workflows**, each responsible for a single feature. A central **Webhook Receiver** routes every incoming update to the correct handler via a Switch node, then all handlers converge back to a single `Respond to Webhook (200 OK)` node.

```
Telegram Update
      │
      ▼
Webhook Receiver  ──► Switch (route by update type)
      │                      │
      │              ┌───────┴────────┐
      │          Handler A       Handler B  ...
      │              │                │
      └──────── Respond to Webhook (200 OK)
```

Each handler is a separate workflow triggered via `Execute Workflow`. They run independently, can be tested individually, and failures are isolated.

**Result:**

| | V1 | V2 |
|---|---|---|
| Workflows | 1 | 23 |
| Nodes total | 485 | ~519 |
| Largest single workflow | 485 | 79 (edit) |
| Webhook response | Separate API call | Inline reply |
| Telegram Trigger node | ✅ | ❌ replaced |
| Independent testing | ❌ | ✅ |
| Isolated failures | ❌ | ✅ |

---

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full workflow reference, data flow diagram, and database schema.

---

## Workflows

```
workflows/
├── core/
│   ├── webhook-receiver.json          # Entry point — routes all Telegram updates
│   └── callback-query.json            # Dispatcher for all inline button presses
│
├── commands/                          # Triggered by text commands
│   ├── start-referrals.json           # /start — registration + referral system
│   ├── menu.json                      # /menu — main keyboard
│   ├── myplaylist.json                # /myplaylist — playlist list
│   ├── edit.json                      # /edit — playlist editing (rename, delete)
│   └── back.json                      # /back — delete message
│
├── callbacks/                         # Triggered by inline button presses
│   ├── get-file.json                  # Deliver audio file to user
│   ├── more-results.json              # Paginate TG mode results
│   ├── dzmore-results.json            # Paginate Pro/Deezer results
│   ├── download-track.json            # Download from Deezer via yt-dlp
│   ├── music-mode-selector.json       # Show current mode + toggle buttons
│   ├── activate-pro.json              # Switch to Deezer Pro mode
│   ├── deactivate-pro.json            # Switch back to TG mode
│   ├── add-to-playlist.json           # Show playlist picker
│   ├── playlist-add.json              # Finalize adding track to playlist
│   └── view-playlist.json             # Show playlist contents
│
├── events/                            # Triggered by non-command updates
│   ├── audio.json                     # Process audio file sent by user
│   ├── user-blocked.json              # Handle bot being blocked
│   └── profile.json                   # Show user profile + stats
│
└── admin/
    ├── admin-commands.json            # /reply_*, /support_*, /admin_* commands
    └── create-playlist.json           # Initiate playlist creation flow
```

---

## Database

PostgreSQL. Full schema in [ARCHITECTURE.md — Database section](./ARCHITECTURE.md#database).

Core tables:

| Table | Purpose |
|---|---|
| `pocketsound_users` | User profiles, ban/admin flags, mode preference, referral data |
| `pocketsound_files` | Indexed audio files — stored as Telegram `message_id` references |
| `pocketsound_file_search_index` | GIN full-text search index over file metadata |
| `pocketsound_playlists` | User playlists |
| `pocketsound_playlist_tracks` | Tracks within playlists |
| `pocketsound_user_states` | FSM state per user (for multi-step flows) |
| `pocketsound_user_limits` | Per-user playlist/track limits |
| `pocketsound_download_queue` | Deezer download queue with locking |
| `pocketsound_tg_chats` | Storage chats for audio files |

---

## Setup

### Requirements

- n8n instance (self-hosted, publicly accessible via HTTPS)
- PostgreSQL 14+
- Server with `yt-dlp` installed (for Deezer, YT downloads)
- Telegram Bot Token from [@BotFather](https://t.me/BotFather)
- A Telegram supergroup for admin/support with Topics enabled

### Steps

**1. Database**

```sql
-- Run the schema file
\i database/schema.sql
```

**2. n8n Credentials**

Create a Postgres credential in n8n (Settings → Credentials) and connect it to your database. All workflow nodes reference this credential by name.

**3. Import workflows**

Import all JSON files from the `workflows/` folder into n8n. Import `webhook-receiver.json` last.

**4. Configure placeholders**

Before importing, replace the following placeholders in all JSON files:

| Placeholder | Where to find it |
|---|---|
| `YOUR_BOT_TOKEN` | @BotFather |
| `YOUR_WEBHOOK_PATH` | Generate a UUID or use any unique string |
| `YOUR_ADMIN_CHAT_ID` | Telegram supergroup chat ID (negative number) |
| `YOUR_INSTANCE_ID` | n8n Settings → n8n API |

Or run this from the `workflows/` directory:

```bash
find . -name "*.json" | xargs sed -i \
  's/YOUR_BOT_TOKEN/YOUR_ACTUAL_TOKEN/g' \
  's/YOUR_WEBHOOK_PATH/YOUR_ACTUAL_PATH/g' \
  's/YOUR_ADMIN_CHAT_ID/YOUR_ACTUAL_CHAT_ID/g'
```

**5. Register webhook with Telegram**

```
https://api.telegram.org/botYOUR_BOT_TOKEN/setWebhook?url=YOUR_N8N_URL/webhook/YOUR_WEBHOOK_PATH
```

**6. Activate**

Activate only the **Webhook Receiver** workflow. All other workflows remain inactive — they are sub-workflows called via `Execute Workflow` and do not need to be active.

---

## Planned — V3

V3 addresses security, duplication, and complexity issues found in V2. See [ARCHITECTURE.md — Planned Architecture V3](./ARCHITECTURE.md#planned-architecture--v3) for the full plan.
