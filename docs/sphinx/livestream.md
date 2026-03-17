# Livestream System — M2Portal

> Based on actual repository code inspection. All references point to real files and implementations.

---

## Overview

M2Portal's livestream system uses **OvenMediaEngine (OME)** for RTMP ingest and HLS/LLHLS playback, **LiveKit** for room lifecycle management via webhooks, **Redis** for chat message storage and moderation state, and **Pusher/Reverb WebSocket** for real-time event broadcasting.

---

## Key Files

| Component | File Path |
|-----------|-----------|
| Routes | `packages/shaunsocial/livestream/routes/api.php` (1037 lines) |
| Livestream Model | `packages/shaunsocial/livestream/src/Models/Livestream.php` |
| Session Model | `packages/shaunsocial/livestream/src/Models/LivestreamSession.php` |
| Chat Controller | `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamChatController.php` (748 lines) |
| Moderation Controller | `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamModerationController.php` (638 lines) |
| Webhook Controller | `packages/shaunsocial/livestream/src/Http/Controllers/Api/LiveKitWebhookController.php` (440 lines) |
| Broadcast Event | `packages/shaunsocial/livestream/src/Events/LivestreamChatBroadcast.php` |
| Notification Job | `packages/shaunsocial/livestream/src/Jobs/SendStreamStartedNotifications.php` |
| Watchdog Command | `packages/shaunsocial/livestream/src/Console/Commands/OmeHlsWatchdog.php` |
| Service Provider | `packages/shaunsocial/livestream/src/ApplicationServiceProvider.php` |

---

## API Routes

### Public Routes (No Auth Required)

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | `/api/livestreams` | Inline Closure | List all live streams (sorted by promotion & viewers) |
| GET | `/api/livestreams/{slug}` | Inline Closure | Get single stream details with viewer count |
| GET | `/api/livestreams/{slug}/chat/history` | `LivestreamChatController@history` | Get last 100 chat messages + pinned message |
| GET | `/api/livestreams/{slug}/chat/state` | `LivestreamModerationController@state` | Get chat settings & moderation state |
| POST | `/api/livestreams/{slug}/viewer/ping` | Inline Closure | Register guest viewer (90s tracking window) |
| POST | `/api/ome/admission` | Inline Closure | OME HLS admission control for RTMP publish |

### Authenticated Routes (Require `auth:sanctum`)

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | `/api/livestreams/check-promotion` | Inline Closure | Check if user's stream is currently promoted |
| GET | `/api/livestreams/mine` | Inline Closure | Get user's own livestream + verification/block status |
| POST | `/api/livestreams/start` | Inline Closure | Start a new livestream |
| POST | `/api/livestreams/{slug}/obs-ingress` | Inline Closure | Get OBS RTMP URL & stream key |
| POST | `/api/livestreams/{slug}/upload-sound` | Inline Closure | Upload custom notification sounds |
| POST | `/api/livestreams/{slug}/chat/send` | `LivestreamChatController@send` | Send chat message |
| POST | `/api/livestreams/{slug}/chat/delete-message` | `LivestreamChatController@deleteMessage` | Delete chat message |
| POST | `/api/livestreams/{slug}/chat/pin-message` | `LivestreamChatController@pinMessage` | Pin message to top |
| POST | `/api/livestreams/{slug}/chat/unpin-message` | `LivestreamChatController@unpinMessage` | Unpin message |
| POST | `/api/livestreams/{slug}/chat/settings` | `LivestreamModerationController@updateSettings` | Update chat/overlay settings |
| GET | `/api/livestreams/{slug}/chat/user-search` | `LivestreamModerationController@userSearch` | Search users for moderation |
| POST | `/api/livestreams/{slug}/chat/moderators` | `LivestreamModerationController@updateModerators` | Add/remove moderators |
| GET | `/api/user/chat-color` | Inline Closure | Get user's custom chat name color |
| POST | `/api/user/chat-color` | Inline Closure | Set user's custom chat name color |
| GET | `/api/livestreams/{slug}/session-stats` | Inline Closure | Get last completed session stats |
| POST | `/api/livestreams/{slug}/update-title` | Inline Closure | Update livestream title |
| POST | `/api/livestreams/{slug}/regenerate-stream-key` | Inline Closure | Regenerate RTMP stream key |
| POST | `/api/livestreams/{slug}/admin-end` | Inline Closure | Force-end stream & ban user (admin only) |
| GET | `/api/livestreams/analytics` | Inline Closure | Get stream analytics with date filtering |
| GET | `/api/livestreams/{slug}/viewer/list` | Inline Closure | Get active viewers list |
| POST | `/api/livestreams/{slug}/viewer/ping-auth` | Inline Closure | Register authenticated viewer |

---

## Stream Lifecycle

### 1. Create / Start

**Endpoint:** `POST /api/livestreams/start`

**Request:**
```json
{
  "title": "Stream Title",
  "highlight": true,
  "notify_followers": true
}
```

**Validation:**
- User must be authenticated and verified (`isVerify()`)
- User must not be `livestream_blocked`
- Title: required, string, max 100 characters
- Highlight costs 1000 MP (24h promotion)
- Notify followers costs 1000 MP

**Process:**
1. Validates user verification status and block status
2. Calculates MP cost (0/1000/2000 based on options)
3. Checks wallet balance via `WalletTransaction::getBalanceByUser()`
4. Generates unique slug from username
5. Creates/updates `Livestream` record via `updateOrCreate()`
6. Calls `ensureOmeSlugAndPlayback()` to set up OME streaming
7. Deducts MP cost via `WalletRepository::send_mass_fund()`
8. Dispatches `SendStreamStartedNotifications` job if notify requested

**Response:**
```json
{
  "data": {
    "id": 1,
    "user_id": 123,
    "title": "Stream Title",
    "slug": "username",
    "room_name": "live_user_123",
    "is_live": true,
    "is_promoted": true,
    "promoted_until": "2026-01-02T00:00:00Z",
    "ingress_stream_key": "a1b2c3d4...",
    "playback_url": "https://ome.example.com/live/a1b2c3d4.../llhls.m3u8",
    "provider": "ome_llhls"
  }
}
```

### 2. OBS Ingress Setup

**Endpoint:** `POST /api/livestreams/{slug}/obs-ingress`

**Process:**
1. Finds user's existing `Livestream` record
2. Validates `OME_RTMP_URL` and `OME_HLS_BASE` configs exist
3. Reuses existing `ingress_stream_key` or generates new 32-hex key
4. Sets `playback_url` to `{OME_HLS_BASE}/live/{stream_key}/llhls.m3u8`

**Response:**
```json
{
  "reused": true,
  "url": "rtmp://ome.example.com/live",
  "stream_key": "a1b2c3d4e5f6789012345678abcdef01",
  "room": "live_user_123",
  "playback_url": "https://ome.example.com/live/a1b2c3d4.../llhls.m3u8"
}
```

**OBS Configuration:**
- Server: RTMP URL from response `url` field
- Stream Key: from response `stream_key` field

### 3. Going Live (RTMP → OME → HLS)

**OME Admission Webhook:** `POST /api/ome/admission`
- Validates RTMP publish request from OBS
- Confirms stream key matches an existing livestream

**LiveKit Webhook Events:**
- `ingress_started` → marks stream `is_live = true`
- `room_started` → marks stream `is_live = true`
- `participant_joined` (publisher) → marks stream `is_live = true`

**On Go-Live:**
- Sets `live_session_start:{stream_id}` timestamp in cache
- Initializes `live_peak_viewers:{stream_id}` counter
- Clears any pending close timeout

### 4. Viewer Flow

**Discovery:** `GET /api/livestreams` — returns all live streams sorted by promotion status and viewer count

**Viewing:**
1. `GET /api/livestreams/{slug}` — get stream details and playback URL
2. Client plays HLS stream from `playback_url`
3. `POST /api/livestreams/{slug}/viewer/ping` or `viewer/ping-auth` — register viewer (heartbeat every 90s)
4. Subscribe to WebSocket channel `livestream.chat.{slug}` for real-time chat

**Token/Auth:** No separate stream token endpoint — playback URL is publicly accessible via HLS. Authentication is only needed for chat participation and moderation.

### 5. Stream Stop / Close

**Automatic Close (Watchdog):**
- `OmeHlsWatchdog` runs `ome:hls-watchdog` artisan command
- Checks each live stream's `playback_url` every 15 seconds
- If HLS playlist unavailable for 60+ seconds → calls `closeStream()`
- Health check: HTTP GET to playback URL, expects status 200 with `#EXTM3U` content

**Manual Close (Admin):**
- `POST /api/livestreams/{slug}/admin-end` — force-ends stream and bans user

**Close Process (`LiveKitWebhookController::closeStream()`):**
1. Records session analytics to `LivestreamSession`
2. Sets `is_live = false`
3. Deletes viewer/publisher heartbeat caches
4. Clears all mute keys from Redis
5. Deletes chat history from Redis
6. Broadcasts `chat_clear` event via WebSocket
7. Calls LiveKit `deleteRoom($room_name)`

### 6. Session Recording

On stream close, `recordSession()` creates a `LivestreamSession` record:

**Metrics Collected:**
| Metric | Source |
|--------|--------|
| `duration_seconds` | `session_end - session_start` (minimum 60s) |
| `unique_viewers` | Redis ZCARD from `live_viewers:{stream_id}` |
| `peak_viewers` | Redis GET from `live_peak_viewers:{stream_id}` |
| `chat_messages` | Redis LLEN from `livestream:chat:history:{stream_id}` |
| `unique_chatters` | Extracted from chat history user.id fields |
| `new_followers` | DB count from `user_follows` in session time range |
| `new_subscribers` | DB count from `user_subscribers` in session time range |
| `tips_amount` | DB SUM from `wallet_transactions` (type=tip, amount>0) |
| `tips_count` | DB COUNT from `wallet_transactions` (type=tip, amount>0) |

---

## Chat System

### Sending Messages

**Endpoint:** `POST /api/livestreams/{slug}/chat/send`

**Request:**
```json
{
  "message": "Hello everyone!",
  "reply_to": {
    "id": "uuid",
    "user": { "name": "User", "id": 1 }
  },
  "client_message_id": "client-uuid"
}
```

**Validation Chain:**
1. Check if chat is `enabled`
2. Check if user is `muted` (Redis key `livestream:chat:mute:{id}:{userId}`)
3. Check `slow_mode` rate limit (Redis key `livestream:chat:slow:{id}:{userId}`)
4. Check `follower_only` mode
5. Check `subscriber_only` mode
6. Check `emoji_only` mode

**Commands:**
- `/clear` — clears all chat (owner/mod only)
- `/susturma "username" minutes` — mutes user for N minutes (mod with permission)

**Broadcast Payload (type: `chat`):**
```json
{
  "id": "uuid-v4",
  "type": "chat",
  "message": "Hello everyone!",
  "deleted": false,
  "ts": "2026-01-01T00:00:00.000Z",
  "reply_to": null,
  "client_message_id": "client-uuid",
  "sender_user_id": "123",
  "user": {
    "id": 123,
    "name": "Display Name",
    "user_name": "username",
    "badges": ["subscriber"],
    "name_color": "#hsl-color",
    "is_verify": true,
    "badge": "badge_type"
  }
}
```

### Chat History

**Endpoint:** `GET /api/livestreams/{slug}/chat/history`

**Response:**
```json
{
  "data": [ /* array of chat message objects */ ],
  "pinned_message": { /* pinned message object or null */ }
}
```

Storage: Redis list `livestream:chat:{livestreamId}` (last 100 messages)

---

## Notifications

**Job:** `SendStreamStartedNotifications`
- Dispatched when stream starts with `notify_followers` option
- Acquires 2-minute lock to prevent duplicates
- Queries all followers with `enable_notify = 1`
- Chunks by 500 followers
- Sends `StreamStartedNotification` to each follower
- Runs on `notifications` queue

---

## Frontend Pages

| Page | File | Purpose |
|------|------|---------|
| Create | `resources/js/pages/livestream_create/index.vue` | Stream setup form |
| Viewer | `resources/js/pages/livestream_show/index.vue` | Full viewer + chat + OBS settings |
| Discovery | `resources/js/pages/livestream/index.vue` | Stream listing |
| Analytics | `resources/js/pages/livestream_analytics/index.vue` | Streamer stats dashboard |
| HUD Overlay | `resources/js/pages/livestream_hud_overlay/index.vue` | OBS browser source overlay |
