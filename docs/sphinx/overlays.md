# Overlay & Alert System — M2Portal

> Based on actual repository code inspection. All references point to real files and implementations.

---

## Overview

M2Portal includes a **livestream HUD overlay** system designed as a browser source for OBS or as an in-page display. It shows real-time chat messages, follow/subscription/tip alerts, and viewer lists. Overlay appearance is customizable per stream via moderation settings.

---

## Key Files

| Component | File Path |
|-----------|-----------|
| HUD Overlay Page | `resources/js/pages/livestream_hud_overlay/index.vue` (23,230 bytes) |
| HUD Bootstrap | `resources/js/pages/livestream_hud_bootstrap/index.vue` (3,726 bytes) |
| Moderation Controller | `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamModerationController.php` |
| Broadcast Event | `packages/shaunsocial/livestream/src/Events/LivestreamChatBroadcast.php` |
| Viewer Page (OBS UI) | `resources/js/pages/livestream_show/index.vue` |

---

## Overlay Types

The HUD overlay (`livestream_hud_overlay/index.vue`) renders the following panels:

| Panel | Data Source | Max Items | Description |
|-------|------------|-----------|-------------|
| **Chat** | WebSocket `type: chat` | 150 messages | Live chat messages with colored usernames |
| **Last Followers** | WebSocket `type: follow` | 20 | Recent followers during stream |
| **Last Subscribers** | WebSocket `type: subscription` | 20 | Recent subscriptions during stream |
| **Last Tips** | WebSocket `type: mp_tip` | 20 | Tips with MP amounts |
| **Registered Viewers** | REST API polling | 50 (paginated) | Authenticated viewers currently watching |

---

## Event Listening

The overlay subscribes to a **single public channel**:

- **Channel:** `livestream.chat.{slug}`
- **Event:** `.livestream.chat.event`
- **Handler:** `onData(data)` dispatches based on `data.type`

### Event Type Dispatch

```javascript
// From livestream_hud_overlay/index.vue
onData(data) {
  switch(data.type) {
    case 'chat':         → adds to messages[] (max 150)
    case 'chat_clear':   → clears all messages[]
    case 'follow':       → adds to liveFollows[] (max 20)
    case 'subscription': → adds to liveSubs[] (max 20)
    case 'mp_tip':       → adds to liveTips[] (max 20)
  }
}
```

### Follow Event Payload

```json
{
  "id": "uuid",
  "type": "follow",
  "ts": "2026-01-01T12:00:00.000Z",
  "user": {
    "id": 456,
    "name": "Follower Name",
    "user_name": "follower"
  }
}
```

### Subscription Event Payload

```json
{
  "id": "uuid",
  "type": "subscription",
  "ts": "2026-01-01T12:00:00.000Z",
  "user": {
    "id": 789,
    "name": "Subscriber Name",
    "user_name": "subscriber"
  }
}
```

### Tip Event Payload

```json
{
  "id": "uuid",
  "type": "mp_tip",
  "ts": "2026-01-01T12:00:00.000Z",
  "amount": 500,
  "user": {
    "id": 321,
    "name": "Tipper Name",
    "user_name": "tipper"
  }
}
```

---

## Overlay Settings

Stored in Redis per stream, managed via `LivestreamModerationController@updateSettings`.

### Alert Toggle Settings

| Setting | Redis Key | Type | Default | Description |
|---------|-----------|------|---------|-------------|
| `ov_tips` | `overlayTips` | boolean | 0 | Show tip alerts |
| `ov_subs` | `overlaySubs` | boolean | 0 | Show subscription alerts |
| `ov_follows` | `overlayFollows` | boolean | 0 | Show follow alerts |
| `ov_sound` | `overlaySound` | boolean | 0 | Enable notification sounds |

### Alert Appearance Settings

| Setting | Redis Key | Type | Default | Description |
|---------|-----------|------|---------|-------------|
| `ov_text_color` | `overlayTextColor` | string | `#2b1f16` | Alert text color |
| `ov_bg_gradient` | `overlayBgGradient` | string | `linear-gradient(135deg, #f4e0c3 0%, #e8d4b5 100%)` | Alert background gradient |
| `ov_border_color` | `overlayBorderColor` | string | `#5e3f2a` | Alert border color |
| `ov_display_duration` | `overlayDisplayDuration` | integer | 6 | Alert display time (1-30 seconds) |

### Chat Appearance Settings

| Setting | Redis Key | Type | Default | Description |
|---------|-----------|------|---------|-------------|
| `chat_text_color` | `chatTextColor` | string | `#ffffff` | Chat text color |
| `chat_bg_gradient` | `chatBgGradient` | string | `linear-gradient(135deg, rgba(0,0,0,0.35) 0%, rgba(0,0,0,0.5) 100%)` | Chat background |
| `chat_border_color` | `chatBorderColor` | string | `rgba(255,255,255,0.1)` | Chat border color |

### Notification Sound Settings

| Setting | Redis Key | Type | Default | Description |
|---------|-----------|------|---------|-------------|
| `sound_follow` | `soundFollow` | string | `""` | Custom follow sound URL |
| `sound_tip` | `soundTip` | string | `""` | Custom tip sound URL |
| `sound_subscribe` | `soundSubscribe` | string | `""` | Custom subscribe sound URL |
| `sound_follow_volume` | `soundFollowVolume` | float | 0.5 | Follow sound volume (0-1) |
| `sound_tip_volume` | `soundTipVolume` | float | 0.5 | Tip sound volume (0-1) |
| `sound_subscribe_volume` | `soundSubscribeVolume` | float | 0.5 | Subscribe sound volume (0-1) |

### Sound Upload

**Endpoint:** `POST /api/livestreams/{slug}/upload-sound`
- Throttled: 30 requests/minute
- Accepts audio file upload for follow, tip, or subscribe sounds
- Stored as custom notification sound URLs

---

## Settings API

### Get State

**Endpoint:** `GET /api/livestreams/{slug}/chat/state`

Returns overlay settings as part of the chat state response:

```json
{
  "chatSettings": {
    "enabled": true,
    "slowMode": 2,
    "emojiOnly": false,
    "followersOnly": false,
    "subscribersOnly": false,
    "overlayTips": true,
    "overlaySubs": true,
    "overlayFollows": true,
    "overlaySound": false,
    "overlayTextColor": "#2b1f16",
    "overlayBgGradient": "linear-gradient(135deg, #f4e0c3 0%, #e8d4b5 100%)",
    "overlayBorderColor": "#5e3f2a",
    "overlayDisplayDuration": 6,
    "chatTextColor": "#ffffff",
    "chatBgGradient": "linear-gradient(135deg, rgba(0,0,0,0.35) 0%, rgba(0,0,0,0.5) 100%)",
    "chatBorderColor": "rgba(255,255,255,0.1)",
    "soundFollow": "",
    "soundTip": "",
    "soundSubscribe": "",
    "soundFollowVolume": 0.5,
    "soundTipVolume": 0.5,
    "soundSubscribeVolume": 0.5
  }
}
```

### Update Settings

**Endpoint:** `POST /api/livestreams/{slug}/chat/settings`

Requires owner or moderator permission. Broadcasts `chat_settings` event via WebSocket after update.

---

## HUD Overlay Features

### Visual Features
- **Draggable header** with overlay mode CSS classes
- **Dynamic username coloring** via HSL color generation from user ID
- **Emoji/emoticon rendering** with sprite sheet support (`m2-sprite` class)
- **Jumbo emoji display** (56×56) when message contains ≤3 emojis only
- **Auto-scroll** to latest messages

### Viewer Polling
- Fetches viewer list every 3 seconds via `GET /api/livestreams/{slug}/viewer/list`
- Displays up to 50 viewers with "load more" pagination (+50 per page)

### Keyboard Shortcuts
| Shortcut | Action |
|----------|--------|
| `Ctrl+Shift+O` | Hide overlay |
| `Ctrl+Shift+Q` | Close overlay |

---

## Not Found / Missing Features

The following overlay features are **not implemented** in the current repository:

| Feature | Status |
|---------|--------|
| Visual overlay editor/builder | ❌ Not found |
| Overlay position/layout customization | ❌ Not found |
| Custom alert animations | ❌ Not found |
| Text-to-speech for tips | ❌ Not found |
| Sub-goal / donation-goal widgets | ❌ Not found |
| Stream labels (latest follower, etc.) | ❌ Not found |
| Poll overlay widget | ❌ Not found (Poll model exists in core but not integrated with livestream overlay) |
| Giveaway overlay widget | ❌ Not found |
