# Tasks — M2P Live Studio

> Tasks grouped by implementation status: existing (already built), adapt (needs adjustment), and new (needs to be built from scratch).

---

## [existing] — Already Implemented

These features are fully functional in the current repository:

### Authentication & Users
- [x] Login via email/username/phone + password — `AuthController@login`
- [x] User registration with configurable fields — `AuthController@signup`
- [x] Two-factor authentication (email, phone, authenticator) — `TwoFactorController`
- [x] Password recovery flow — `UserController@send_code_forgot_password`
- [x] Laravel Sanctum token management — `PersonalAccessToken` with Redis caching
- [x] User verification system — required for streaming
- [x] RBAC roles and permissions — `Role`, `Permission`, `RolePermission` models
- [x] reCAPTCHA spam protection on login/signup
- [x] Rate limiting on login endpoint — `throttle:login`

### Livestream Core
- [x] Stream creation with title — `POST /api/livestreams/start`
- [x] RTMP ingest via OvenMediaEngine — `POST /api/livestreams/{slug}/obs-ingress`
- [x] Stream key generation (32-hex) — `Livestream@ensureOmeSlugAndPlayback()`
- [x] Stream key regeneration — `POST /api/livestreams/{slug}/regenerate-stream-key`
- [x] HLS/LLHLS playback URL generation — `{OME_HLS_BASE}/live/{key}/llhls.m3u8`
- [x] LiveKit webhook for stream lifecycle — `LiveKitWebhookController@handle`
- [x] OME admission webhook — `POST /api/ome/admission`
- [x] HLS watchdog (auto-close after 60s) — `OmeHlsWatchdog` artisan command
- [x] Stream close with full cleanup — `LiveKitWebhookController@closeStream()`
- [x] Stream listing sorted by promotion & viewers — `GET /api/livestreams`
- [x] Stream details API — `GET /api/livestreams/{slug}`
- [x] Viewer heartbeat tracking (90s window) — `POST /api/livestreams/{slug}/viewer/ping`
- [x] Viewer list API — `GET /api/livestreams/{slug}/viewer/list`
- [x] Stream promotion (24h highlight for 1000 MP) — in start route
- [x] Update stream title — `POST /api/livestreams/{slug}/update-title`

### Livestream Chat
- [x] Send chat messages with validation — `LivestreamChatController@send`
- [x] Chat history (last 100 from Redis) — `LivestreamChatController@history`
- [x] Chat message broadcasting — `LivestreamChatBroadcast` event
- [x] Message pinning with 24h TTL — `LivestreamChatController@pinMessage`
- [x] Message unpinning — `LivestreamChatController@unpinMessage`
- [x] Message deletion (owner/mod) — `LivestreamChatController@deleteMessage`
- [x] Chat clear command (`/clear`) — in `send()` method
- [x] Mute command (`/susturma "name" minutes`) — in `send()` method
- [x] Reply threading — `reply_to` field in payload
- [x] User badges (owner, moderator, subscriber) — in `send()` method
- [x] Deterministic name colors from user ID — `nameColorFromId()`
- [x] Custom chat name color — `POST /api/user/chat-color`
- [x] Emoji support with sprite sheet rendering

### Moderation
- [x] Chat enable/disable — `updateSettings()` with `enabled` field
- [x] Slow mode (0-600s) — `updateSettings()` with `slowMode` field
- [x] Follower-only mode — `updateSettings()` with `followersOnly` field
- [x] Subscriber-only mode — `updateSettings()` with `subscribersOnly` field
- [x] Emoji-only mode — `updateSettings()` with `emojiOnly` field
- [x] Add/remove moderators — `LivestreamModerationController@updateModerators`
- [x] Granular moderator permissions — `canDeleteMessage`, `canMuteUser`
- [x] User search for moderation — `LivestreamModerationController@userSearch`
- [x] Admin force-end stream + ban — `POST /api/livestreams/{slug}/admin-end`
- [x] Chat state API (public) — `LivestreamModerationController@state`
- [x] Settings broadcast via WebSocket — `chat_settings` event type

### Overlay & Alerts
- [x] HUD overlay browser source — `livestream_hud_overlay/index.vue`
- [x] Chat panel in overlay — max 150 messages
- [x] Follow alert panel — max 20 recent followers
- [x] Subscription alert panel — max 20 recent subscribers
- [x] Tip alert panel — max 20 tips with MP amounts
- [x] Viewer list panel — polled every 3s
- [x] Overlay appearance customization — text/bg/border colors
- [x] Alert display duration (1-30s) — configurable per stream
- [x] Custom notification sounds — upload via `POST /api/livestreams/{slug}/upload-sound`
- [x] Sound volume control (0-1) — per sound type
- [x] Keyboard shortcuts (Ctrl+Shift+O/Q) — hide/close overlay

### Monetization
- [x] MP wallet system — `WalletTransaction`, `WalletOrder`, `WalletPackage`
- [x] Tipping during livestreams — `mp_tip` event type
- [x] Subscription system — `UserSubscriber`, `SubscriberPackage`
- [x] Paid content — `UserPostPaid`, `UserPostPaidOrder`
- [x] Payment gateways — Stripe, PayPal, Razorpay, Binance, +5 more
- [x] Wallet top-up and withdrawal

### Analytics
- [x] Session recording on stream close — `LivestreamSession` model
- [x] Peak/average/unique viewers — Redis-based tracking
- [x] Chat message count — from Redis history
- [x] Tips amount and count — from wallet transactions
- [x] New followers/subscribers — from database
- [x] Analytics dashboard — `livestream_analytics/index.vue`
- [x] Analytics API with date filtering — `GET /api/livestreams/analytics`

### Notifications
- [x] Stream-start push notifications — `SendStreamStartedNotifications` job
- [x] Firebase FCM integration — `kreait/laravel-firebase`

### Frontend Pages
- [x] Stream creation page — `livestream_create/index.vue`
- [x] Stream viewer page with OBS settings — `livestream_show/index.vue` (203KB)
- [x] Stream discovery page — `livestream/index.vue`
- [x] Analytics dashboard — `livestream_analytics/index.vue`
- [x] HUD overlay page — `livestream_hud_overlay/index.vue`
- [x] Full chat UI — `resources/js/pages/chat/`

---

## [adapt] — Needs Adjustment

These features exist but need modification for M2P Live Studio:

### Architecture Refactoring
- [ ] Extract route closure logic to `LivestreamController` class
  - *Affected: `routes/api.php` lines 75-1037 (multiple inline closures)*
  - *Benefit: testability, reusability, cleaner code*
- [ ] Create `LivestreamService` class for business logic
  - *Extract: stream creation, ingress setup, analytics from closures/controllers*
- [ ] Create `IngestService` class for OME integration
  - *Extract: RTMP URL construction, stream key management, admission validation*
- [ ] Create `AnalyticsService` for metric aggregation
  - *Extract: session recording logic from `LiveKitWebhookController@recordSession`*

### Livestream Model Enhancements
- [ ] Add `category` field to `Livestream` model
  - *Currently: title-only identification*
- [ ] Add `tags` field (JSON) to `Livestream` model
  - *For: search and discovery*
- [ ] Add `language` field to `Livestream` model
  - *For: international discovery*

### Overlay Persistence
- [ ] Persist overlay settings to database
  - *Currently: Redis-only (lost on restart)*
  - *Create: `LivestreamOverlaySettings` model*
- [ ] Allow multiple overlay themes per user
  - *Currently: single settings per stream*

### Chat Persistence
- [ ] Add optional MySQL persistence for chat messages
  - *Currently: Redis-only (deleted on stream close)*
  - *For: VOD chat replay feature*

### Poll Integration
- [ ] Integrate existing `Poll` model with livestream overlay
  - *Poll, PollItem, PollVote models exist in core package*
  - *Need: livestream-specific poll creation/display/voting routes*

### Streamer Dashboard
- [ ] Unify stream setup + OBS settings + moderation into single dashboard
  - *Currently: spread across livestream_create, livestream_show, livestream_analytics*

### Discovery Enhancement
- [ ] Add category/tag filtering to stream listing
  - *Currently: sorted by promotion + viewer count only*

---

## [new] — Needs to Be Built

These features don't exist in the repository and must be built from scratch:

### Stream Recording & VOD
- [ ] OME recording integration or separate recording service
- [ ] `StreamRecording` model (id, livestream_id, user_id, url, duration, size, created_at)
- [ ] VOD playback page
- [ ] VOD management API (list, delete, update metadata)
- [ ] Chat replay for VODs (paired with chat persistence)
- [ ] Thumbnail generation from VOD content

### Clip System
- [ ] `StreamClip` model (id, recording_id, user_id, start_time, end_time, title, url)
- [ ] Clip creation API (from live or VOD)
- [ ] Clip viewer page
- [ ] Clip sharing and embedding

### Game Category System
- [ ] `StreamCategory` model (id, name, slug, icon, game_count)
- [ ] `StreamTag` model (id, name, slug)
- [ ] Category CRUD API
- [ ] Game directory browse page
- [ ] Category-based search and filtering

### Stream Scheduling
- [ ] `StreamSchedule` model (id, user_id, title, category_id, scheduled_at, description)
- [ ] Schedule creation/update API
- [ ] Schedule listing page
- [ ] Scheduled event notifications to followers
- [ ] Calendar view

### Advanced Engagement
- [ ] Channel points/loyalty system
  - `ChannelPoint` model, earn rules, redemption system
- [ ] Predictions system
  - `Prediction` model, voting, resolution, payout
- [ ] Raid system
  - Send viewers to another live stream
- [ ] Host system
  - Rebroadcast another stream on your channel

### Advanced Overlay Features
- [ ] Visual overlay editor/builder UI (drag-and-drop)
- [ ] Custom alert animations (CSS keyframes / Lottie)
- [ ] Giveaway overlay widget (entry, countdown, winner)
- [ ] Stream labels widget (latest follower, top tipper, etc.)
- [ ] Sub-goal / donation-goal progress bars
- [ ] Text-to-speech for tip messages

### Stream Quality Monitoring
- [ ] Real-time bitrate monitoring dashboard
- [ ] Frame drop detection and alerts
- [ ] Viewer buffering analytics
- [ ] Multi-bitrate encoding configuration
- [ ] Adaptive bitrate playback setup

### Custom Emotes & Badges
- [ ] `ChannelEmote` model (id, user_id, code, image_url, tier)
- [ ] Emote upload and management API
- [ ] Custom subscriber badge tiers
- [ ] Emote rendering in chat overlay

---

## Task Priority Summary

| Priority | Category | Count | Description |
|----------|----------|-------|-------------|
| 🔴 P0 | [adapt] Architecture | 4 tasks | Service layer, controller extraction |
| 🔴 P0 | [adapt] Model enhancements | 3 tasks | Category, tags, language fields |
| 🟡 P1 | [new] Game categories | 4 tasks | Discovery and browsing |
| 🟡 P1 | [adapt] Overlay persistence | 2 tasks | Settings durability |
| 🟡 P1 | [adapt] Dashboard | 1 task | Unified streamer experience |
| 🟡 P2 | [new] VOD/Recording | 6 tasks | Content longevity |
| 🟡 P2 | [new] Clips | 4 tasks | Content sharing |
| 🟡 P2 | [adapt] Poll integration | 1 task | Viewer engagement |
| 🟢 P3 | [new] Scheduling | 5 tasks | Planning and notifications |
| 🟢 P3 | [new] Advanced overlays | 6 tasks | Professional tools |
| 🟢 P3 | [new] Engagement | 4 tasks | Viewer retention |
| 🟢 P4 | [new] Quality monitoring | 5 tasks | Professional streaming |
| 🟢 P4 | [new] Custom emotes | 4 tasks | Community building |
