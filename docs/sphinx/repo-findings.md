# Repository Findings — M2Portal

> Auto-generated from actual repository inspection. All file paths, class names, routes, and data structures are verified from source code.

---

## 1. Authentication Flow

**Technology:** Laravel Sanctum (token-based API authentication)

**Key Files:**
- Controller: `packages/shaunsocial/core/src/Http/Controllers/Api/AuthController.php`
- Routes: `packages/shaunsocial/core/routes/api.php`
- Config: `config/auth.php`, `config/sanctum.php`
- User Model: `packages/shaunsocial/core/src/Models/User.php`
- Token Model: `packages/shaunsocial/core/src/Models/Sanctum/PersonalAccessToken.php`
- Middleware: `packages/shaunsocial/core/src/Http/Middleware/ApplicationApi.php`

**Flow:**
1. `POST /api/auth/login` → validates email/username/phone + password → returns Sanctum token
2. If two-factor enabled, returns error with `two_factory_code` → user calls `POST /api/auth/login_with_code`
3. `POST /api/auth/signup` → creates user → returns Sanctum token
4. All protected routes use `auth:sanctum` middleware
5. Token stored via `$user->createToken('authToken')->plainTextToken`
6. `PersonalAccessToken` overrides default Sanctum with Redis caching and custom pruning

**Guards:** `web` (session), `admin` (session) — both using Eloquent `User` provider

---

## 2. Livestream Flow

**Technology:** OvenMediaEngine (OME) for RTMP ingest + HLS/LLHLS playback; LiveKit for room lifecycle webhooks

**Key Files:**
- Routes: `packages/shaunsocial/livestream/routes/api.php` (1037 lines, 23 routes)
- Models: `packages/shaunsocial/livestream/src/Models/Livestream.php`, `LivestreamSession.php`
- Controllers: `LivestreamChatController.php`, `LivestreamModerationController.php`, `LiveKitWebhookController.php`
- Event: `packages/shaunsocial/livestream/src/Events/LivestreamChatBroadcast.php`
- Job: `packages/shaunsocial/livestream/src/Jobs/SendStreamStartedNotifications.php`
- Watchdog: `packages/shaunsocial/livestream/src/Console/Commands/OmeHlsWatchdog.php`

**Lifecycle:**
1. **Create/Start:** `POST /api/livestreams/start` — validates verified user, charges MP for highlight/notifications, creates `Livestream` record with OME slug and playback URL
2. **OBS Ingress:** `POST /api/livestreams/{slug}/obs-ingress` — returns RTMP URL + stream key for OBS configuration
3. **Go Live:** LiveKit webhook (`ingress_started` / `room_started`) sets `is_live = true`
4. **Viewer:** `GET /api/livestreams/{slug}` returns stream data; playback via `{OME_HLS_BASE}/live/{stream_key}/llhls.m3u8`
5. **Stop:** `OmeHlsWatchdog` monitors playback URL every 15s, closes stream after 60s of unavailability; or admin force-end via `POST /api/livestreams/{slug}/admin-end`
6. **Session Recording:** On close, `LiveKitWebhookController::closeStream()` records analytics to `LivestreamSession`

---

## 3. Ingest Logic

**RTMP Ingest via OvenMediaEngine:**
- RTMP URL configured via `config('services.ome_rtmp_url')`
- HLS base URL configured via `config('services.ome_hls_base')`
- Stream key: 32-character hex string (`bin2hex(random_bytes(16))`)
- Playback URL format: `{OME_HLS_BASE}/live/{stream_key}/llhls.m3u8`
- OME admission webhook: `POST /api/ome/admission` validates RTMP publish

**LiveKit Integration:**
- Room name format: `live_user_{userId}`
- LiveKit SDK: `agence104/livekit-server-sdk` v1.3
- Used for: room lifecycle management, webhook-based stream start/stop detection
- LiveKit does NOT handle video/chat for livestreaming (only for voice/video calls)

---

## 4. Chat System

### Livestream Chat (Real-time)
**Technology:** Redis-backed storage + WebSocket broadcasting via Pusher/Reverb

**Key Files:**
- Controller: `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamChatController.php` (748 lines)
- Event: `packages/shaunsocial/livestream/src/Events/LivestreamChatBroadcast.php`
- Channel: `livestream.chat.{slug}` (public, no auth required)

**Features:**
- Chat history: last 100 messages stored in Redis (`livestream:chat:{livestreamId}`)
- Message pinning with 24h TTL
- Commands: `/clear` (clear chat), `/susturma "name" minutes` (mute user — Turkish for "muting")
- User badges: `owner`, `moderator`, `subscriber`
- Name colors: deterministic HSL from user ID, or custom via `/api/user/chat-color`
- Reply threading via `reply_to` field

### DM/Group Chat
**Technology:** Eloquent models + WebSocket broadcasting

**Key Files:**
- Controller: `packages/shaunsocial/chat/src/Http/Controllers/Api/ChatController.php`
- Models: `ChatRoom.php`, `ChatMessage.php`, `ChatMessageItem.php`, `ChatRoomMember.php`
- Events: `MessageSentEvent.php`, `MessageUnsentEvent.php`, `RoomSeenEvent.php`, etc.
- Channel: `Social.Chat.Models.ChatRoom.{id}` (private, auth required)

---

## 5. Overlay System

**Key Files:**
- Frontend: `resources/js/pages/livestream_hud_overlay/index.vue` (23,230 bytes)
- Bootstrap: `resources/js/pages/livestream_hud_bootstrap/index.vue`
- Moderation Controller: `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamModerationController.php` (overlay settings)

**Overlay Types:**
1. **Chat Panel** — real-time chat messages with user colors
2. **Follow Alerts** — recent followers list (max 20)
3. **Subscription Alerts** — recent subscriptions (max 20)
4. **Tip Alerts** — monetary tips with MP amounts (max 20)
5. **Viewer List** — authenticated viewers (polled every 3s)

**Overlay Settings (stored in Redis per stream):**
- `ov_tips`, `ov_subs`, `ov_follows`, `ov_sound` — enable/disable each alert type
- `ov_text_color`, `ov_bg_gradient`, `ov_border_color` — visual customization
- `ov_display_duration` — alert display time (1-30 seconds)
- `sound_follow`, `sound_tip`, `sound_subscribe` — custom sound URLs
- `sound_follow_volume`, `sound_tip_volume`, `sound_subscribe_volume` — sound volumes (0-1)
- `chat_text_color`, `chat_bg_gradient`, `chat_border_color` — chat appearance

**Event Listening:**
- Channel: `livestream.chat.{slug}` (public)
- Event: `.livestream.chat.event`
- Types processed: `chat`, `chat_clear`, `follow`, `subscription`, `mp_tip`

---

## 6. Moderation System

**Key Files:**
- Controller: `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamModerationController.php` (638 lines)
- Chat Controller: `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamChatController.php`
- DB Table: `livestream_moderators`

**Features:**
- **Mute Users:** Redis-backed temporary mute (`livestream:chat:mute:{livestreamId}:{userId}`)
- **Slow Mode:** Rate limiting per user (`livestream:chat:slow:{livestreamId}:{userId}`), configurable 0-600s
- **Chat Modes:** `follower_only`, `subscriber_only`, `emoji_only`, `enabled` (enable/disable entire chat)
- **Moderator Roles:** Owner can add/remove moderators with granular permissions (`canDeleteMessage`, `canMuteUser`)
- **Message Deletion:** Owner or moderator can delete messages
- **Message Pinning:** Owner, moderator, or admin can pin/unpin messages
- **Chat Clear:** `/clear` command clears all messages (owner/mod only)
- **Admin Force-End:** `POST /api/livestreams/{slug}/admin-end` — ends stream + bans user

---

## 7. Frontend Integration

**Technology:** Vue 3.5 + Vite 4.0 + PrimeVue 4.3 + Tailwind CSS 3.1

**Livestream Pages:**
| Page | File | Size | Purpose |
|------|------|------|---------|
| Discovery | `resources/js/pages/livestream/index.vue` | 17KB | Stream list with "YAYINA DÖN (OBS)" button |
| Viewer | `resources/js/pages/livestream_show/index.vue` | 203KB | Full viewer with video player, chat, OBS settings, moderation |
| Create | `resources/js/pages/livestream_create/index.vue` | 13KB | Stream setup form (title, highlight, notify) |
| Analytics | `resources/js/pages/livestream_analytics/index.vue` | 20KB | Streamer analytics dashboard |
| HUD Overlay | `resources/js/pages/livestream_hud_overlay/index.vue` | 23KB | OBS browser source overlay |
| HUD Bootstrap | `resources/js/pages/livestream_hud_bootstrap/index.vue` | 4KB | Overlay bootstrap/init |

**Key Frontend Libraries:**
- `hls.js` — HLS video playback
- `ovenplayer` — OvenMediaEngine player integration
- `livekit-client` — LiveKit client SDK (for voice/video calls, not livestream)
- `laravel-echo` + `pusher-js` — WebSocket real-time communication

**Mobile App:** Flutter/Dart at `mobile/shaun_app/`

---

## 8. Missing Parts for M2P Live Studio

Based on repository analysis, the following are **not found** or need development:

| Feature | Status | Notes |
|---------|--------|-------|
| Dedicated OBS settings page | ⚠️ Partial | Embedded within `livestream_show/index.vue`, not standalone |
| Poll system (livestream) | ❌ Not found | `Poll` model exists in core but not integrated with livestream |
| Giveaway system | ❌ Not found | No giveaway model or controller found |
| Stream recording/VOD | ❌ Not found | Sessions recorded but no video archiving |
| Multi-bitrate encoding | ❌ Not found | Single HLS stream via OME |
| Raid/host functionality | ❌ Not found | No raid or host system |
| Clip creation | ❌ Not found | No clip functionality |
| Overlay editor/builder | ❌ Not found | Overlay settings exist but no visual editor |
| Stream schedule | ❌ Not found | No scheduling system |
| Chat replay for VOD | ❌ Not found | Chat stored only in Redis (ephemeral) |
| Stream categories/tags | ⚠️ Partial | Title only, no category system for discovery |
| Emote/badge system | ⚠️ Partial | Badges exist (owner/mod/sub) but no custom emotes |
| Channel points | ❌ Not found | No loyalty/points system for viewers |
| Dedicated streamer dashboard | ⚠️ Partial | Analytics page exists, but no unified dashboard |
