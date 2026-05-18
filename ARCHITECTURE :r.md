# PocketSound Bot — n8n Workflow Architecture

> Telegram music bot built on n8n. Handles audio search, Deezer Pro mode, playlist management, referral system, and admin tooling. All logic is split into isolated sub-workflows triggered by a central Webhook Receiver.

---

## Table of Contents

- [Current Architecture — V2](#current-architecture--v2)
  - [Data Flow](#data-flow)
  - [Workflow Reference](#workflow-reference)
- [Database](#database)
- [Planned Architecture — V3](#planned-architecture--v3)
  - [What Changes](#what-changes)
  - [New Workflows](#new-workflows)
  - [Refactored Workflows](#refactored-workflows)
  - [Removed Duplication](#removed-duplication)

---

## Current Architecture — V2

### Data Flow

```
Telegram
   │
   ▼
Webhook Receiver  ──────────────────────────────────────────────────────┐
   │  (single entry point, routes all updates)                          │
   │                                                                    │
   ├── /start, ref start ──► Handler: /start & Referrals               │
   ├── /menu             ──► Handler: /menu                             │
   ├── /myplaylist       ──► Handler: /myplaylist                       │
   ├── /edit             ──► Handler: /edit                             │
   ├── /back             ──► Handler: /back                             │
   ├── text message      ──► Handler: text_search                       │
   ├── audio file        ──► Handler: audio                             │
   ├── admin command     ──► Handler: admin_commands                    │
   ├── my_chat_member    ──► Handler: user_blocked                      │
   ├── Profile button    ──► Handler: Profile                           │
   ├── Create Playlist   ──► Handler: Create Playlist                   │
   ├── View Playlist     ──► Handler: View Playlist                     │
   └── callback_query    ──► Handler: callback_query                    │
                                  │                                     │
                                  ├──► Handler: get_file                │
                                  ├──► Handler: more_results            │
                                  ├──► Handler: dzmore_results          │
                                  ├──► Handler: download_track          │
                                  ├──► Handler: music_mode_selector     │
                                  ├──► Handler: Activate PRO Music Mode │
                                  ├──► Handler: /deactivate PRO Music Mode
                                  ├──► Handler: add_to_playlist         │
                                  ├──► Handler: View Playlist           │
                                  └──► Handler: Playlist Add            │
                                                                        │
All handlers respond 200 OK via Webhook Receiver ◄──────────────────────┘
```

Every handler is an n8n sub-workflow (`executeWorkflowTrigger`). The Webhook Receiver stays active; all handlers run with `active: false` and are invoked via `Execute Workflow` node.

---

### Workflow Reference

#### Webhook Receiver
**ID:** `OvaMQAjAI501Pj5M` | **Nodes:** 18

Single entry point for all Telegram updates. Receives the raw webhook payload, routes it via a Switch node to the correct handler based on update type (message, callback_query, my_chat_member, audio). Responds `200 OK` immediately after dispatching.

Routes:
- Text commands (`/start`, `/menu`, `/edit`, `/back`, `/myplaylist`) → dedicated handlers
- Free text → `text_search`
- Audio file → `audio`
- `callback_query` → `callback_query` dispatcher
- Admin commands (`/reply_*`, `/support_*`, `/admin_*`) → `admin_commands`
- `my_chat_member` (bot blocked) → `user_blocked`
- Profile / View Playlist buttons → respective handlers

---

#### Handler: /start & Referrals
**ID:** `XLO8fLnJJ2PSUThR` | **Nodes:** 50

Handles `/start` command for both direct launches and referral deep links (`?start=ref_XXXX`).

- Checks if user is banned or admin mode is active
- Detects whether it's a new user or returning user
- **New user:** upserts to `pocketsound_users`, generates referral link, initializes coin balance, creates a dedicated support topic in admin group
- **Returning user:** updates profile fields (username, name, premium status)
- **Referral flow:** validates the referral code, prevents self-referral, credits the referrer, updates `referred_by` in DB
- Sends welcome message to user, notifies admin group

DB tables touched: `pocketsound_users`, `pocketsound_coin_balances`

---

#### Handler: /menu
**ID:** `TaqwqSzotuemlTMu` | **Nodes:** 12

Displays the main menu keyboard to the user after the welcome screen.

- Verifies user is not banned and admin mode is not active
- Sends the main menu message with reply keyboard buttons
- Notifies admin group

---

#### Handler: /myplaylist
**ID:** `rWidkVpII2dp6oS7` | **Nodes:** 14

Shows the user's list of playlists as inline keyboard buttons.

- Checks ban/admin state
- Queries `pocketsound_playlists` for user's playlists
- Formats results as inline buttons (each opens View Playlist)
- Shows "no playlists" message if empty
- Notifies admin group

---

#### Handler: /edit
**ID:** `v6AHpLe6hOvIuabJ` | **Nodes:** 79

Monolithic handler for all playlist editing operations. Routes internally via a Switch node based on callback data prefix.

Handles five distinct flows:
1. **List playlists for editing** — shows playlists with Edit action buttons
2. **Rename playlist** — sets a `rename` state in DB, waits for next text message to apply the new name
3. **Delete playlist** — shows confirmation prompt, handles confirm/cancel
4. **List tracks for deletion** — fetches playlist tracks, sends each as audio message with a Delete button
5. **Delete track** — removes specific track from playlist, confirms to user

DB tables touched: `pocketsound_playlists`, `pocketsound_playlist_tracks`, `pocketsound_users` (state management)

---

#### Handler: /back
**ID:** `rdPUDL1iGiKJv2Sl` | **Nodes:** 12

Handles the Back button. Deletes the message the button was attached to (via `deleteMessage`) so the UI stays clean.

- Checks ban/admin state
- Calls Telegram `deleteMessage` API
- Notifies admin group

---

#### Handler: text_search
**ID:** `u8GcZtPcfaKgA2G3` | **Nodes:** 49

Central handler for all free-text messages. Determines the intent based on user state stored in DB.

Three distinct paths:
1. **Playlist naming** — if user has `awaiting_name` state set (triggered by Create Playlist), saves the playlist name and clears state
2. **Playlist renaming** — if user has `rename` state, applies new name to the playlist and clears state
3. **Music search** — searches `pocketsound_files` in TG mode, or calls Deezer API in Pro mode. Returns results as inline keyboard buttons. Saves query to search history.

Also handles the Deezer API call and result formatting when Pro mode is active.

DB tables touched: `pocketsound_users`, `pocketsound_playlists`, `pocketsound_files`, `pocketsound_search_history`

---

#### Handler: audio
**ID:** `WzrThWlK6JuHD5yC` | **Nodes:** 15

Processes audio files sent directly by users.

- Validates user is not banned
- Inserts TG chat record and audio file metadata into DB
- Updates full-text search index
- Sends the audio back to the user with an "Add to Playlist" inline button
- Notifies admin group with file details

DB tables touched: `pocketsound_files`, `pocketsound_chats`

---

#### Handler: callback_query
**ID:** `AtwKGveF3eHcLWmY` | **Nodes:** 30

Dispatcher for all `callback_query` updates (inline button presses).

- Checks ban and admin mode once for all callbacks
- Parses `callback_data` prefix and routes to the correct sub-workflow:
  - `get|*` → `Handler: get_file`
  - `more|*` → `Handler: more_results`
  - `dzmore|*` → `Handler: dzmore_results`
  - `dz|*` → `Handler: download_track`
  - `music_mode` → `Handler: music_mode_selector`
  - `activate_pro` → `Handler: Activate PRO Music Mode`
  - `deactivate_pro` → `Handler: /deactivate PRO Music Mode`
  - `add_to_playlist|*` → `Handler: add_to_playlist`
  - `playlist_view|*` → `Handler: View Playlist`
  - `playlist_add|*` → `Handler: Playlist Add`
  - Unknown → sends "unknown button" message to user

---

#### Handler: get_file
**ID:** `dds2h41obxWvwXTG` | **Nodes:** 13

Delivers a stored audio file to the user when they tap a search result button.

- Parses file ID from callback data
- Looks up `tg_chat_id` and `message_id` in `pocketsound_files` table
- Uses Telegram `copyMessage` to forward the file to the user with an "Add to Playlist" button
- Notifies admin group

DB tables touched: `pocketsound_files`

---

#### Handler: more_results
**ID:** `b3M0xJJUvE33i7YC` | **Nodes:** 16

Paginates search results in Telegram (TG) mode.

- Parses query and page offset from callback data
- Queries `pocketsound_files` with pagination
- Builds inline keyboard with next batch of results
- Sends updated keyboard to user

---

#### Handler: dzmore_results
**ID:** `gKfLxxhv4nOn0Yk7` | **Nodes:** 16

Paginates search results in Deezer (Pro) mode.

- Parses query and page index from callback data
- Calls Deezer search API with offset
- Normalizes track titles and artist names
- Builds inline keyboard with next batch of Deezer results
- Sends updated keyboard to user

---

#### Handler: download_track
**ID:** `kNiZacNPYT7TvoSy` | **Nodes:** 52

Downloads a track from Deezer (Pro mode) and delivers it to the user.

- Checks if track already exists in `pocketsound_files` (cache hit → send existing file immediately)
- Checks download queue length — if server is busy, notifies user and queues the request
- Calls Deezer API to resolve track URL
- Downloads audio file to server via shell command (`yt-dlp` or equivalent)
- Fetches album artwork, prepares thumbnail
- Sends audio file to admin storage chat (to get a Telegram `file_id`)
- Saves file metadata and `file_id` to DB, indexes for search
- Forwards file to user with "Add to Playlist" button
- Cleans up temporary files from server

DB tables touched: `pocketsound_files`, `pocketsound_download_queue`

---

#### Handler: music_mode_selector
**ID:** `ebTgmHPRvHC0EJhU` | **Nodes:** 19

Shows the current music search mode and buttons to switch between TG mode and Pro (Deezer) mode.

- Queries `is_api_mode` flag from user record
- Sends a status message with toggle buttons depending on current mode
- Notifies admin group

---

#### Handler: Activate PRO Music Mode
**ID:** `Wl2f0OvDsBlYp0RT` | **Nodes:** 15

Switches the user's search mode to Deezer Pro.

- Updates `is_api_mode = true` in `pocketsound_users`
- Deletes the mode selector message
- Sends confirmation message to user
- Notifies admin group

---

#### Handler: /deactivate PRO Music Mode
**ID:** `RQIVT8H3U7kb7pgW` | **Nodes:** 15

Switches the user's search mode back to Telegram (standard) mode.

- Updates `is_api_mode = false` in `pocketsound_users`
- Deletes the mode selector message
- Sends confirmation message to user
- Notifies admin group

---

#### Handler: Create Playlist
**ID:** `ApZ3GRdZ3kp6KdOX` | **Nodes:** 19

Initiates playlist creation.

- Checks if user has reached the playlist limit
- Sets `awaiting_name` state in user record
- Sends a prompt asking the user to type the playlist name
- The actual playlist is created in `text_search` when the user sends the name

DB tables touched: `pocketsound_users` (state), `pocketsound_playlists` (limit check)

---

#### Handler: View Playlist
**ID:** `Qp9SbwhXYbQSyVPo` | **Nodes:** 17

Displays the contents of a specific playlist.

- Fetches playlist name and all tracks from DB
- Formats track list as an inline keyboard (each track is a button that triggers `get_file`)
- Sends the playlist view message to user
- Notifies admin group

DB tables touched: `pocketsound_playlists`, `pocketsound_playlist_tracks`

---

#### Handler: add_to_playlist
**ID:** `u0WvMyrtlf9mDDHm` | **Nodes:** 22

Shows the user a list of their playlists to choose where to add a track.

- Checks user has at least one playlist and that not all are full
- Fetches non-full playlists from DB
- Formats them as inline keyboard buttons
- Sends the playlist picker to user

DB tables touched: `pocketsound_playlists`

---

#### Handler: Playlist Add
**ID:** `E5sELYlxEaPil8zC` | **Nodes:** 27

Finalizes adding a track to a selected playlist.

- Checks the selected playlist is not full (max 33 tracks)
- Checks the track is not already in the playlist
- Inserts the track into `pocketsound_playlist_tracks`
- Sends confirmation message to user
- Notifies admin group

DB tables touched: `pocketsound_playlist_tracks`, `pocketsound_playlists`

---

#### Handler: Profile
**ID:** `mOVWWx2U5Z4R4byY` | **Nodes:** 14

Shows the user's profile and usage statistics.

- Queries aggregated stats: playlists count, tracks saved, searches performed, referrals made, coin balance
- Formats and sends the profile message
- Notifies admin group

---

#### Handler: admin_commands
**ID:** `y6ShAPjvBgbVxWk9` | **Nodes:** 15

Handles commands sent by admins from the support group.

- Validates sender is an authorized admin
- Routes based on command prefix:
  - `/reply_*` → sends a message to a specific user
  - `/support_*` → posts a reply in the user's support thread
  - `/admin_true` → enables admin mode (bot goes into maintenance, blocks all regular users)
  - `/admin_false` → disables admin mode

---

#### Handler: user_blocked
**ID:** `87zNCHRgfe190GoL` | **Nodes:** 7

Handles `my_chat_member` events when a user blocks the bot.

- Detects `new_status = kicked` transition
- Sets `is_bot_kicked = true` in user record
- Notifies admin group in the user's support thread

DB tables touched: `pocketsound_users`

---

## Database

PostgreSQL. Tables are grouped by domain. The **Bot Core** group is required for the bot to function. **MiniApp** tables are used by the web mini-application and are not required for bot operation.

---

### Bot Core Tables

#### `pocketsound_users`
Central user record. Created on first `/start`.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | Internal user ID |
| `tg_id` | BIGINT UNIQUE | Telegram user ID. Nullable for web-only users |
| `tg_username` | VARCHAR(64) | |
| `tg_first_name` | VARCHAR(128) | |
| `tg_last_name` | VARCHAR(128) | |
| `photo_url` | TEXT | |
| `premium` | BOOLEAN | Telegram Premium status |
| `language_code` | VARCHAR(8) | |
| `balance_stars` | NUMERIC(12,4) | Telegram Stars balance |
| `balance_wallet` | NUMERIC(12,4) | Internal wallet balance |
| `user_role` | VARCHAR(32) | Default `'user'` |
| `is_banned` | BOOLEAN | Ban flag |
| `ban_reason` | TEXT | |
| `banned_at` | TIMESTAMP | |
| `admin_active` | BOOLEAN | Temporary freeze — user cannot interact with bot while admin dialog is open |
| `support_topic_id` | BIGINT | Thread ID in admin support group, created on first `/start` |
| `referrer_id` | BIGINT | Internal `id` of user who referred this user |
| `referral_link` | TEXT | Personal deep link for referrals |
| `is_bot_kicked` | BOOLEAN | Set to TRUE when user blocks the bot |
| `is_api_mode` | BOOLEAN | TRUE = Pro/Deezer mode, FALSE = TG mode |
| `email` | VARCHAR(255) | MiniApp — web login email |
| `phone` | VARCHAR(20) | MiniApp |
| `is_email_confirmed` | BOOLEAN | MiniApp |
| `last_active_at` | TIMESTAMP | |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

Indexes: `tg_id`, `support_topic_id`, `referrer_id`, `email`

---

#### `pocketsound_tg_chats`
Telegram chats used as file storage. The bot stores audio files in a private admin channel to obtain persistent Telegram `file_id` values.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `tg_chat_id` | BIGINT UNIQUE | Telegram chat ID |
| `chat_type` | VARCHAR(32) | `private`, `channel`, `group` |
| `title` | VARCHAR(255) | |
| `username` | VARCHAR(64) | |
| `is_public` | BOOLEAN | |
| `is_indexable` | BOOLEAN | Whether files from this chat are included in search |

---

#### `pocketsound_files`
Every indexed audio file. Stores Telegram location (`tg_chat_id` + `message_id`) rather than raw bytes — files live in Telegram storage.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | Internal file ID used in all callback data |
| `owner_user_id` | BIGINT FK → users | Uploader |
| `tg_chat_id` | BIGINT FK → tg_chats | Storage chat |
| `message_id` | INT | Message in storage chat — used with `copyMessage` |
| `file_unique_id` | VARCHAR(255) UNIQUE | Telegram's dedup key |
| `file_type` | VARCHAR(32) | `audio`, `voice`, etc. |
| `mime_type` | VARCHAR(128) | |
| `title` | VARCHAR(255) | |
| `artist` | VARCHAR(255) | |
| `album` | VARCHAR(255) | |
| `file_name` | VARCHAR(255) | |
| `duration` | INT | Seconds |
| `file_size` | BIGINT | Bytes |
| `is_public` | BOOLEAN | |
| `is_searchable` | BOOLEAN | Included in text search |

Indexes: `(tg_chat_id, message_id)`, `is_searchable`

---

#### `pocketsound_file_search_index`
Full-text search index over file metadata. Populated when a file is uploaded or downloaded.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `file_id` | BIGINT FK → files | |
| `search_text` | TEXT | Raw text (title + artist + album) |
| `normalized_text` | TEXT | Lowercased, transliterated, stripped |

Index: GIN `to_tsvector('simple', normalized_text)` — enables full-text search

---

#### `pocketsound_playlists`
User-created playlists.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `user_id` | BIGINT FK → users | Owner |
| `name` | VARCHAR(100) | |
| `description` | TEXT | |
| `is_public` | BOOLEAN | Default FALSE |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

Indexes: `user_id`, `is_public` (partial, where TRUE)

---

#### `pocketsound_playlist_tracks`
Junction table — one row per track in a playlist. Enforces uniqueness per playlist.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `playlist_id` | BIGINT FK → playlists | |
| `file_id` | BIGINT FK → files | |
| `position` | INTEGER | Track order |
| `added_at` | TIMESTAMP | |

Unique constraint: `(playlist_id, file_id)`
Indexes: `playlist_id`, `(playlist_id, position)`, `file_id`

---

#### `pocketsound_user_limits`
Per-user limits for playlists and tracks. Allows purchased upgrades.

| Column | Type | Notes |
|---|---|---|
| `user_id` | BIGINT PK FK → users | |
| `max_playlists` | INTEGER | Default 3 |
| `extra_playlists_purchased` | INTEGER | Paid upgrades |
| `max_tracks_per_playlist` | INTEGER | Default 33 |
| `extra_slots_purchased` | INTEGER | Paid upgrades |

---

#### `pocketsound_user_states`
State machine for multi-step flows (playlist creation, rename). One active state per user at a time.

| Column | Type | Notes |
|---|---|---|
| `user_id` | BIGINT PK FK → users | |
| `current_state` | VARCHAR(64) | e.g. `awaiting_name`, `rename` |
| `state_data` | JSONB | Arbitrary context (e.g. playlist_id being renamed) |
| `state_started_at` | TIMESTAMP | |

Indexes: `user_id`, `current_state`, `(user_id, current_state)` partial where not null

---

#### `pocketsound_search_queries`
Search history per user.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `user_id` | BIGINT FK → users | |
| `query_text` | TEXT | |
| `results_count` | INT | |
| `created_at` | TIMESTAMP | |

---

#### `pocketsound_download_queue`
Queue for Deezer track downloads. Handles concurrency — if the server is busy, requests are queued.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `user_id` | BIGINT FK → users | |
| `chat_id` | BIGINT | User's chat ID |
| `track_id` | VARCHAR(64) | Deezer track ID |
| `track_title` | VARCHAR(255) | |
| `track_artist` | VARCHAR(255) | |
| `deezer_track_id` | BIGINT | |
| `stream_url` | TEXT | Resolved audio stream URL |
| `songlink_youtube_url` | TEXT | YouTube fallback URL |
| `cover_url` | TEXT | Album art URL |
| `output_path` | VARCHAR(255) | Temp file path on server |
| `support_topic_id` | BIGINT | For admin notifications |
| `status` | VARCHAR(20) | `pending` → `processing` → `done` / `failed` |
| `file_unique_id` | VARCHAR(255) | Telegram file_unique_id after upload |
| `message_id` | BIGINT | Telegram message ID after upload |
| `sent_to_user` | BOOLEAN | Whether file was delivered |
| `locked_at` | TIMESTAMP | Optimistic lock timestamp |
| `locked_by` | TEXT | Lock owner (worker ID) |
| `error_message` | TEXT | |
| `created_at` | TIMESTAMP | |
| `started_at` | TIMESTAMP | |
| `finished_at` | TIMESTAMP | |

Indexes: `status`, `user_id`

---

#### `pocketsound_file_shares`
Tracks file sharing events between users.

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `file_id` | BIGINT FK → files | |
| `from_user_id` | BIGINT FK → users | |
| `to_user_id` | BIGINT FK → users | |
| `share_type` | VARCHAR(32) | |

---

### Creator & Social Tables

Used by the creator/social layer. Not required for core bot functionality.

| Table | Purpose |
|---|---|
| `pocketsound_creators` | Creator profiles — bio, verified flag, aggregated stats |
| `pocketsound_stories` | Short-form content posted by creators (expiring) |
| `pocketsound_streams` | Live and VOD streams |
| `pocketsound_ads` | Ad campaigns — budget, targeting, schedule |
| `pocketsound_ad_stories` | Junction: ads attached to stories |
| `pocketsound_transactions` | All financial events (Stars payments, gifts) with JSONB metadata |
| `pocketsound_playlist_deezer_tracks` | Deezer track IDs saved to playlists (Pro mode) |
| `pocketsound_user_likes` | User ↔ Deezer track likes |
| `pocketsound_user_history` | Listening history per user (Deezer track IDs) |

---

### MiniApp Auth Tables

Used exclusively by the web Mini Application. Not touched by bot workflows.

| Table | Purpose |
|---|---|
| `pocketsound_credentials` | Web login credentials (hashed password, lockout tracking) |
| `pocketsound_auth_codes` | Email confirmation and password reset codes |
| `pocketsound_sessions` | JWT refresh token sessions |

---

### Entity Relationship (simplified)

```
pocketsound_users
  ├── pocketsound_user_limits          (1:1)
  ├── pocketsound_user_states          (1:1, active FSM state)
  ├── pocketsound_search_queries       (1:N)
  ├── pocketsound_download_queue       (1:N)
  ├── pocketsound_playlists            (1:N)
  │     └── pocketsound_playlist_tracks  (N:M via file_id)
  │           └── pocketsound_files
  │                 ├── pocketsound_tg_chats     (storage location)
  │                 └── pocketsound_file_search_index
  ├── pocketsound_file_shares          (1:N as sender/receiver)
  ├── pocketsound_creators             (1:1, optional)
  │     ├── pocketsound_stories
  │     └── pocketsound_streams
  └── pocketsound_transactions         (1:N)

pocketsound_ads
  └── pocketsound_ad_stories  ──► pocketsound_stories

[MiniApp only]
pocketsound_credentials  ──► pocketsound_users
pocketsound_auth_codes   ──► pocketsound_users
pocketsound_sessions     ──► pocketsound_users
```

---

## Planned Architecture — V3

### New Workflows

| Workflow | Purpose |
|---|---|
| `Guard: check_user` | Reusable ban/admin check. Called at the start of every handler. Returns `{ allowed, user, reason }`. |
| `Helper: register_user` | Reusable new-user registration. Upsert, referral link, coin init, support topic creation. |
| `Handler: Edit — List` | Browse playlists for editing (split from `/edit`) |
| `Handler: Edit — Rename` | Rename a playlist (split from `/edit`) |
| `Handler: Edit — Delete` | Delete playlist or individual track (split from `/edit`) |

--- 

### V3 Planned

pocketsound-bot/
│
├── README.md                         
├── ARCHITECTURE.md                    
├── CHANGELOG.md                      
│
├── workflows/
│   ├── core/
│   │   ├── webhook-receiver.json
│   │   ├── callback-query.json
│   │   └── text-search.json
│   │
│   ├── commands/                      
│   │   ├── start-referrals.json
│   │   ├── menu.json
│   │   ├── back.json
│   │   ├── myplaylist.json
│   │   └── edit.json
│   │
│   ├── callbacks/                     
│   │   ├── get-file.json
│   │   ├── more-results.json
│   │   ├── dzmore-results.json
│   │   ├── download-track.json
│   │   ├── music-mode-selector.json
│   │   ├── activate-pro.json
│   │   ├── deactivate-pro.json
│   │   ├── add-to-playlist.json
│   │   ├── playlist-add.json
│   │   └── view-playlist.json
│   │
│   ├── events/                        
│   │   ├── audio.json
│   │   ├── user-blocked.json
│   │   └── profile.json
│   │
│   └── admin/
│       ├── admin-commands.json
│       └── create-playlist.json
│
├── database/
│   ├── schema.sql                    
│   ├── migrations/
│   │   ├── 001_initial.sql
│   │   ├── 002_add_ban_fields.sql
│   │   ├── 003_add_referrals.sql
│   │   └── ...
│   └── seed/
│       └── example_data.sql           
│
├── docs/
│   ├── setup.md                      
│   ├── n8n-variables.md               
│   └── admin-commands.md              
│
└── .env.example                       
