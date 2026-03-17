# Roadmap — M2P Live Studio

> Based on repository analysis of existing vs. missing features.

---

## Phase 0 — Foundation (Already Exists ✅)

These features are **already implemented** in the M2Portal repository and form the foundation:

- [x] Laravel 12 backend with modular package system
- [x] Vue 3 + Vite frontend with Tailwind CSS
- [x] Flutter mobile app
- [x] Laravel Sanctum authentication (login, signup, 2FA)
- [x] User verification system (required for streaming)
- [x] RBAC (roles, permissions, admin/moderator hierarchy)
- [x] RTMP ingest via OvenMediaEngine → HLS/LLHLS playback
- [x] Stream key management (generate, regenerate, reuse)
- [x] OBS integration (RTMP URL + stream key endpoint)
- [x] Stream lifecycle (create → start → live → stop)
- [x] HLS watchdog (auto-close after 60s of unavailability)
- [x] LiveKit webhook integration for room lifecycle
- [x] Redis-backed livestream chat (100 message history)
- [x] WebSocket broadcasting (Pusher/Reverb)
- [x] Chat features (reply, pin, delete, clear)
- [x] Chat commands (`/clear`, `/susturma`)
- [x] User badges (owner, moderator, subscriber)
- [x] Custom chat name colors
- [x] Moderation (mute, slow mode, follower-only, subscriber-only, emoji-only)
- [x] Moderator roles with granular permissions
- [x] Admin force-end stream + ban
- [x] Browser source HUD overlay (chat, follows, subs, tips, viewers)
- [x] Overlay customization (colors, gradients, borders, durations)
- [x] Notification sounds (custom sound URLs with volume control)
- [x] MP wallet system (in-app currency)
- [x] Tipping during livestreams
- [x] Creator subscriptions (tiered)
- [x] Payment gateways (Stripe, PayPal, Razorpay, +6 more)
- [x] Session analytics (viewers, chat, tips, followers, subscribers)
- [x] Analytics dashboard with date filtering
- [x] Stream-start push notifications to followers
- [x] Stream promotion (24h highlight for 1000 MP)
- [x] Viewer tracking (heartbeat-based, 90s window)
- [x] DM/Group chat with voice calls

---

## Phase 1 — Architecture Improvements (Adapt)

Refactor existing code to support M2P Live Studio features:

- [ ] Extract route closure business logic into dedicated controller methods
  - *Currently: ~500 lines of business logic in `routes/api.php` inline closures*
  - *Target: `LivestreamController` with proper methods*
- [ ] Create service layer for livestream package
  - *Add: `LivestreamService`, `IngestService`, `AnalyticsService`*
- [ ] Add `category` and `tags` fields to `Livestream` model
  - *Currently: title-only stream identification*
- [ ] Persist overlay settings to database (currently Redis-only)
  - *Allow: reusable overlay themes across sessions*
- [ ] Add chat message persistence option
  - *Currently: Redis-only (ephemeral)*
  - *Target: optional MySQL persistence for VOD chat replay*

---

## Phase 2 — Game Discovery & Categories (New)

Build game/content discovery system:

- [ ] Create `StreamCategory` model (game categories)
- [ ] Create `StreamTag` model (tags per stream)
- [ ] Add category selection to stream creation flow
- [ ] Build game directory page (browse by category)
- [ ] Add tag-based search and filtering
- [ ] Update stream listing with category filters

---

## Phase 3 — Stream Recording & VOD (New)

Add video-on-demand capabilities:

- [ ] Integrate OME recording or external recording service
- [ ] Create `StreamRecording` model (VOD metadata)
- [ ] Build VOD playback page
- [ ] Add chat replay for VODs (requires Phase 1 chat persistence)
- [ ] Add clip creation from live or VOD content
- [ ] Create `StreamClip` model
- [ ] Build clip viewer and sharing

---

## Phase 4 — Enhanced Overlay System (Adapt + New)

Improve the overlay and alert experience:

- [ ] Build visual overlay editor/builder UI
- [ ] Add overlay position and layout customization
- [ ] Add custom alert animations (CSS/Lottie)
- [ ] Build poll overlay widget
  - *Note: `Poll` model exists in core but not integrated with livestream*
- [ ] Build giveaway overlay widget
- [ ] Add stream labels (latest follower, top tipper, etc.)
- [ ] Add sub-goal / donation-goal progress bars
- [ ] Add text-to-speech for tip messages

---

## Phase 5 — Advanced Engagement (New)

Build viewer engagement features:

- [ ] Channel points / loyalty system
  - *Viewers earn points for watching, redeem for rewards*
- [ ] Predictions (viewers bet on outcomes)
- [ ] Raid functionality (send viewers to another stream)
- [ ] Host functionality (rebroadcast another stream)
- [ ] Stream scheduling with calendar
- [ ] Schedule notifications to followers

---

## Phase 6 — Streamer Dashboard (Adapt)

Unify existing tools into a dedicated streamer experience:

- [ ] Build unified streamer dashboard page
  - *Combine: stream setup + OBS settings + analytics + moderation*
- [ ] Add real-time stream quality metrics (bitrate, frame drops)
- [ ] Add revenue analytics (tips, subs, donations over time)
- [ ] Add audience demographics view
- [ ] Add growth tracking and milestones
- [ ] Add stream manager view (multi-panel: chat, viewers, events)

---

## Phase 7 — Advanced Streaming (New)

Add professional streaming features:

- [ ] Multi-bitrate encoding support via OME
- [ ] Adaptive bitrate playback
- [ ] Low-latency mode toggle
- [ ] Stream health monitoring in real-time
- [ ] Custom RTMP server configuration per user
- [ ] Multi-platform simulcast (optional)

---

## Phase 8 — Community & Social (Adapt)

Leverage existing social features for streaming:

- [ ] Integrate Portalk (Discord-like) communities with stream channels
- [ ] Add stream chat integration with group chat
- [ ] Build streamer community pages with subscriber-only content
- [ ] Add custom emote/badge system per channel
- [ ] Integrate stories for stream highlights and teasers

---

## Priority Matrix

| Priority | Phase | Effort | Value |
|----------|-------|--------|-------|
| 🔴 High | Phase 1 — Architecture | Medium | Enables all future work |
| 🔴 High | Phase 2 — Discovery | Medium | Core user experience |
| 🟡 Medium | Phase 3 — VOD | High | Retention & content longevity |
| 🟡 Medium | Phase 4 — Overlays | Medium | Streamer satisfaction |
| 🟡 Medium | Phase 6 — Dashboard | Medium | Streamer productivity |
| 🟢 Low | Phase 5 — Engagement | High | Viewer retention |
| 🟢 Low | Phase 7 — Advanced | High | Professional streamers |
| 🟢 Low | Phase 8 — Community | Medium | Ecosystem integration |
