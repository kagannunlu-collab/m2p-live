# Product — M2P Live Studio

> Game streaming tool built on top of the M2Portal platform.

---

## Vision

**M2P Live Studio** is a game streaming tool that leverages M2Portal's existing social platform infrastructure to provide streamers with a complete broadcasting, engagement, and monetization toolkit — tailored for the gaming community.

---

## Existing Platform Capabilities (from repository)

The M2Portal repository already provides substantial infrastructure for a streaming platform:

### ✅ Livestream System
- **RTMP Ingest via OvenMediaEngine** — streamers connect OBS to an RTMP server, content is transcoded to HLS/LLHLS for viewers
- **Stream Key Management** — 32-char hex stream keys, key regeneration, per-user streams
- **Stream Lifecycle** — create, start, go live (webhook-based), stop (watchdog-based automatic detection)
- **OBS Integration** — `POST /api/livestreams/{slug}/obs-ingress` returns RTMP URL + stream key
- **Playback** — `{OME_HLS_BASE}/live/{stream_key}/llhls.m3u8` low-latency HLS playback
- **Promotion** — 24-hour stream highlighting for 1000 MP

*Key files: `packages/shaunsocial/livestream/routes/api.php`, `packages/shaunsocial/livestream/src/Models/Livestream.php`*

### ✅ Real-time Chat
- **Redis-backed livestream chat** — last 100 messages, message pinning, reply threads
- **WebSocket broadcasting** via Pusher/Reverb on public channel `livestream.chat.{slug}`
- **User badges** — owner, moderator, subscriber
- **Custom name colors** — deterministic HSL or user-selected
- **Chat commands** — `/clear`, `/susturma "name" minutes`
- **Emoji support** with sprite sheet rendering and jumbo emoji display

*Key files: `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamChatController.php`*

### ✅ Moderation
- **User muting** — time-based, Redis-backed
- **Slow mode** — configurable 0-600 second rate limiting
- **Chat modes** — follower-only, subscriber-only, emoji-only, chat enable/disable
- **Moderator roles** — granular permissions (`canDeleteMessage`, `canMuteUser`)
- **Message management** — pin, unpin, delete messages
- **Admin force-end** — end stream + ban user

*Key files: `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamModerationController.php`*

### ✅ Overlay / HUD System
- **Browser source overlay** — `livestream_hud_overlay/index.vue` for OBS
- **Panels:** Chat, Followers, Subscribers, Tips, Viewers
- **Customizable appearance** — text colors, backgrounds, borders per stream
- **Notification sounds** — custom sounds for follow, tip, subscribe events
- **Keyboard shortcuts** — Ctrl+Shift+O (hide), Ctrl+Shift+Q (close)

*Key files: `resources/js/pages/livestream_hud_overlay/index.vue`*

### ✅ Monetization
- **MP Wallet System** — in-app currency with top-up and withdrawal
- **Tips** — send MP during livestreams, tip packages configurable
- **Subscriptions** — tiered subscriptions to creators
- **Paid Content** — exclusive posts behind paywall
- **Payment Gateways** — Stripe, PayPal, Razorpay, Binance, Cashfree, Flutterwave, Paystack, NowPayments, CCBill

*Key files: `packages/shaunsocial/paid_content/`, `packages/shaunsocial/wallet/`, `packages/shaunsocial/gateway/`*

### ✅ Analytics
- **Session recording** — peak/avg/unique viewers, chat count, tips, followers, subscribers
- **Analytics dashboard** — `livestream_analytics/index.vue` with date filtering
- **Viewer tracking** — heartbeat-based concurrent viewers with 90s window

*Key files: `packages/shaunsocial/livestream/src/Models/LivestreamSession.php`, `resources/js/pages/livestream_analytics/index.vue`*

### ✅ Social Features
- **User profiles** — follow, subscribe, block, report
- **DM/Group chat** — with voice calls, file sharing, fund transfers
- **Groups** — community groups with moderation
- **Stories** — ephemeral content with music and backgrounds
- **Short-form video (Vibb)** — TikTok-like short videos
- **Discord-like communities (Portalk)** — guilds, channels, voice, reactions

### ✅ Authentication & Users
- **Laravel Sanctum** — token-based API auth
- **Two-factor auth** — email, phone, authenticator app
- **User verification** — required for streaming
- **RBAC** — roles, permissions, moderator/admin hierarchy

### ✅ Push Notifications
- **Firebase FCM** — push notifications to mobile
- **Stream-start notifications** — notify followers when going live (costs 1000 MP)

---

## M2P Live Studio — Target Feature Set

Building on the existing platform, M2P Live Studio adds:

### 🎮 Game-Specific Features
- Stream categories (game selection)
- Game tags and discovery
- Game-specific directory pages

### 🎬 Advanced Streaming
- Multi-bitrate encoding support
- Stream recording / VOD
- Clip creation from live streams
- Stream scheduling

### 🎨 Enhanced Overlays
- Visual overlay editor/builder
- Custom alert animations
- Poll overlay widget
- Giveaway overlay widget
- Stream labels (latest follower, top tipper, etc.)
- Sub-goal / donation-goal progress bars

### 🏆 Engagement
- Channel points / loyalty system
- Predictions (viewer bets on outcomes)
- Raids (send viewers to another stream)
- Chat replay for VODs

### 📊 Advanced Analytics
- Revenue analytics (tips, subs, donations over time)
- Stream quality metrics (bitrate, frame drops)
- Audience demographics
- Growth tracking

---

## Target Users

| User Type | Needs | Platform Features |
|-----------|-------|-------------------|
| **Casual Streamer** | Easy setup, basic chat | Stream creation, OBS integration, chat |
| **Growing Streamer** | Monetization, engagement | Tips, subscriptions, overlays, analytics |
| **Professional Streamer** | Full control, analytics | Advanced overlays, moderation, multi-bitrate |
| **Viewer** | Watch, interact, support | HLS playback, chat, tipping, following |
| **Moderator** | Chat management | Mute, slow mode, chat modes, message management |
