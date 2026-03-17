# Architecture — M2Portal

> Based on actual repository code inspection. All references point to real files and implementations.

---

## System Overview

M2Portal is a **social media platform** with integrated **livestreaming**, **monetization**, **real-time chat**, and **Discord-like community** features. It's built as a monorepo containing a Laravel backend, Vue.js frontend, Flutter mobile app, and a modular package system.

---

## Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| **Backend** | Laravel (PHP) | 12.x (PHP 8.2+) |
| **Frontend** | Vue.js + Vite | Vue 3.5, Vite 4.0 |
| **Mobile** | Flutter/Dart | Dart SDK |
| **UI Framework** | PrimeVue + Tailwind CSS | PrimeVue 4.3, Tailwind 3.1 |
| **Database** | MySQL/MariaDB | Eloquent ORM |
| **Cache/Queue** | Redis | Used for sessions, queues, chat, moderation |
| **WebSocket** | Pusher / Laravel Reverb | Real-time broadcasting |
| **Video Streaming** | OvenMediaEngine (OME) | RTMP ingest → HLS/LLHLS playback |
| **Room Management** | LiveKit | Webhook-based room lifecycle, voice/video calls |
| **File Storage** | S3-compatible / Local | Via `StorageService` model |
| **Payments** | Stripe, PayPal, Razorpay, Binance, + 5 more | Via gateway package |
| **Push Notifications** | Firebase (FCM) | Via `kreait/laravel-firebase` |
| **SMS** | Twilio | For phone verification and 2FA |
| **Image Processing** | Intervention Image | v3.2 |
| **Video Processing** | PHP-FFmpeg | v1.1 |

---

## Repository Structure

```
m2portal/
├── app/                          # Laravel application
│   ├── Events/Discord/           # 17 Discord/Portalk broadcast events
│   ├── Http/Controllers/         # App-level controllers (Discord)
│   ├── Models/                   # App models (User, Discord/)
│   └── Providers/                # Service providers
│
├── packages/shaunsocial/         # Modular packages (core business logic)
│   ├── core/                     # Auth, users, posts, social, RBAC
│   ├── livestream/               # Livestream system (OME, chat, moderation)
│   ├── chat/                     # DM/Group chat system
│   ├── paid_content/             # Tips, subscriptions, paid posts
│   ├── wallet/                   # Wallet & payment transactions
│   ├── gateway/                  # Payment gateway integrations
│   ├── group/                    # Community groups
│   ├── story/                    # Stories (ephemeral content)
│   ├── vibb/                     # Short-form video
│   ├── advertising/              # Ad platform
│   ├── user_page/                # Creator pages
│   ├── user_subscription/        # Subscription plans
│   ├── user_verify/              # User verification
│   ├── chatbot/                  # AI chatbot
│   ├── ai_features/              # AI-powered features
│   └── ai_provider/              # AI provider management
│
├── resources/js/pages/           # Vue.js frontend pages (50+ components)
│   ├── livestream/               # Stream discovery
│   ├── livestream_show/          # Stream viewer (203KB — main viewing experience)
│   ├── livestream_create/        # Stream setup
│   ├── livestream_analytics/     # Streamer analytics
│   ├── livestream_hud_overlay/   # OBS browser source overlay
│   ├── chat/                     # DM/Group chat UI
│   └── ...                       # Profile, settings, groups, etc.
│
├── mobile/shaun_app/             # Flutter mobile application
│   ├── lib/app/                  # Dart application code
│   ├── android/                  # Android platform
│   └── ios/                      # iOS platform
│
├── config/                       # Laravel configuration
├── routes/                       # Web, API, admin, broadcast channels
├── database/                     # Migrations, seeders, factories
└── tests/                        # Test suite
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENTS                                     │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │
│  │ Vue SPA  │  │ Flutter  │  │   OBS    │  │ Browser Source   │    │
│  │ (Vite)   │  │  Mobile  │  │ Studio   │  │  (HUD Overlay)   │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘    │
└───────┼──────────────┼─────────────┼─────────────────┼──────────────┘
        │              │             │                 │
        │ HTTP/WS      │ HTTP/WS     │ RTMP            │ HTTP/WS
        ▼              ▼             ▼                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       BACKEND (Laravel 12)                           │
│                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │
│  │   Sanctum   │  │  Reverb/    │  │    Package System            │  │
│  │   Auth      │  │  Pusher     │  │                              │  │
│  │  (Tokens)   │  │  (WebSocket)│  │  core │ livestream │ chat    │  │
│  └─────────────┘  └─────────────┘  │  paid_content │ wallet      │  │
│                                     │  gateway │ group │ story    │  │
│  ┌─────────────┐  ┌─────────────┐  │  advertising │ user_page    │  │
│  │    Redis    │  │   MySQL     │  │  user_subscription │ vibb   │  │
│  │ (Cache/     │  │ (Eloquent   │  │  chatbot │ ai_features      │  │
│  │  Queue/     │  │  ORM)       │  └─────────────────────────────┘  │
│  │  Chat)      │  │             │                                    │
│  └─────────────┘  └─────────────┘                                    │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      │ Webhooks / API calls
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SERVICES                                 │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │ OvenMedia    │  │   LiveKit    │  │   Payment Gateways       │   │
│  │ Engine       │  │   Server     │  │   (Stripe, PayPal,       │   │
│  │ (RTMP→HLS)  │  │   (Rooms/    │  │    Razorpay, Binance,    │   │
│  │              │  │    Voice)    │  │    Cashfree, etc.)        │   │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘   │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │  Firebase    │  │   Twilio     │  │   AI Providers           │   │
│  │  (FCM Push)  │  │   (SMS)      │  │   (Chatbot, Features)    │   │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: Livestream

```
OBS Studio                OvenMediaEngine              M2Portal Backend              Client (Vue/Flutter)
    │                          │                            │                              │
    │  RTMP Stream             │                            │                              │
    │ ─────────────────────>   │                            │                              │
    │                          │  Admission Webhook         │                              │
    │                          │ ─────────────────────────> │ POST /api/ome/admission      │
    │                          │                            │  Validate stream key          │
    │                          │  <─────────────── OK ───── │                              │
    │                          │                            │                              │
    │                          │  Transcode RTMP → HLS      │                              │
    │                          │  Serve .m3u8 playlist      │                              │
    │                          │                            │                              │
    │                          │                            │  LiveKit Webhook              │
    │                          │                            │  (ingress_started)            │
    │                          │                            │  → is_live = true             │
    │                          │                            │                              │
    │                          │                            │                    GET /api/livestreams/{slug}
    │                          │                            │ <──────────────────────────── │
    │                          │                            │ ──── stream data ──────────> │
    │                          │                            │                              │
    │                          │ <──────── HLS Request ─────┼──────────────────────────── │
    │                          │ ──────── .m3u8/.ts ────────┼──────────────────────────── │
    │                          │                            │                              │
    │                          │                            │        WebSocket Subscribe    │
    │                          │                            │ <──── livestream.chat.{slug}  │
    │                          │                            │                              │
    │                          │                            │  POST chat/send              │
    │                          │                            │ <──────────────────────────── │
    │                          │                            │  Broadcast event             │
    │                          │                            │ ──────────────────────────── │
```

---

## Data Flow: Authentication

```
Client                    Laravel Backend                Redis Cache
  │                            │                            │
  │  POST /api/auth/login      │                            │
  │ ─────────────────────────> │                            │
  │                            │  Validate credentials       │
  │                            │  createToken('authToken')   │
  │                            │ ──────────── cache token ─> │
  │  { user, token }           │                            │
  │ <───────────────────────── │                            │
  │                            │                            │
  │  GET /api/resource         │                            │
  │  Authorization: Bearer ... │                            │
  │ ─────────────────────────> │                            │
  │                            │ ──── lookup by SHA256 ──── │
  │                            │ <──── token + user data ── │
  │  { data }                  │                            │
  │ <───────────────────────── │                            │
```

---

## Package Architecture

Each package follows the same structure:

```
packages/shaunsocial/{package_name}/
├── src/
│   ├── Models/               # Eloquent models
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Api/          # API controllers
│   │   │   └── Admin/        # Admin panel controllers
│   │   ├── Requests/         # Form request validation
│   │   └── Middleware/        # Package-specific middleware
│   ├── Events/               # Broadcasting events
│   ├── Jobs/                 # Queue jobs
│   ├── Console/Commands/     # Artisan commands
│   └── ApplicationServiceProvider.php
├── routes/
│   ├── api.php               # API routes
│   ├── web.php               # Web routes
│   └── admin.php             # Admin routes
├── database/
│   └── migrations/           # Database migrations
├── resources/
│   └── views/                # Blade templates
└── config/
    └── constant.php          # Package configuration
```

---

## Integration Plan for M2P Live Studio

### What Already Exists

The M2Portal repository already provides a solid foundation for a game streaming tool:

1. **Stream Infrastructure** — OME RTMP ingest, HLS playback, stream key management
2. **Real-time Chat** — Redis-backed with WebSocket broadcasting, moderation, badges
3. **Overlay System** — HUD overlay as browser source with chat, follow, sub, tip panels
4. **Monetization** — Tips (MP currency), subscriptions, wallet system
5. **Analytics** — Session recording with viewer counts, chat stats, tips, followers
6. **Moderation** — Mute, slow mode, chat modes, moderator roles, message management
7. **Push Notifications** — Stream-start notifications to followers

### Integration Approach

To build M2P Live Studio on top of M2Portal:

1. **Extend the livestream package** — add game-specific features (game category, stream tags, clip creation) to the existing `packages/shaunsocial/livestream/` package
2. **Enhance the overlay system** — add visual overlay editor, custom animations, poll/giveaway widgets
3. **Add stream recording/VOD** — integrate OME recording or add a separate recording service
4. **Build dedicated streamer dashboard** — unify existing analytics + moderation + settings into one page
5. **Add discovery features** — game categories, tags, recommended streams
6. **Leverage existing infrastructure** — reuse auth, wallet, subscriptions, push notifications, chat

### Recommended Architecture Changes

| Area | Current | Proposed |
|------|---------|----------|
| Route closures | Business logic in `routes/api.php` closures | Extract to dedicated controller methods |
| Service layer | No services in livestream package | Add `LivestreamService`, `IngestService` for reusability |
| Chat persistence | Redis only (ephemeral) | Add optional MySQL persistence for VOD chat replay |
| Stream categories | Title only | Add `category` and `tags` fields to `Livestream` model |
| Overlay config | Redis-only | Persist overlay themes to database for reuse |
