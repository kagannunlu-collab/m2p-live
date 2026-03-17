# Chat System — M2Portal

> Based on actual repository code inspection. All references point to real files and implementations.

---

## Overview

M2Portal has two distinct chat systems:

1. **Livestream Chat** — Redis-backed, real-time broadcast via public WebSocket channel
2. **DM/Group Chat** — Eloquent model-backed, private WebSocket channels per room

---

## 1. Livestream Chat

### Key Files

| Component | File Path |
|-----------|-----------|
| Chat Controller | `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamChatController.php` |
| Broadcast Event | `packages/shaunsocial/livestream/src/Events/LivestreamChatBroadcast.php` |
| Moderation Controller | `packages/shaunsocial/livestream/src/Http/Controllers/Api/LivestreamModerationController.php` |
| HUD Overlay | `resources/js/pages/livestream_hud_overlay/index.vue` |
| Viewer Page | `resources/js/pages/livestream_show/index.vue` |

### WebSocket Channel

- **Channel:** `livestream.chat.{slug}` (public — no authorization required)
- **Event Name:** `livestream.chat.event`
- **Implementation:** `LivestreamChatBroadcast` implements `ShouldBroadcastNow`

### API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/livestreams/{slug}/chat/history` | No | Get last 100 messages + pinned message |
| GET | `/api/livestreams/{slug}/chat/state` | No | Get chat settings, moderation state, viewer permissions |
| POST | `/api/livestreams/{slug}/chat/send` | Yes | Send chat message |
| POST | `/api/livestreams/{slug}/chat/delete-message` | Yes | Delete a message (owner/mod) |
| POST | `/api/livestreams/{slug}/chat/pin-message` | Yes | Pin message to top (owner/mod/admin) |
| POST | `/api/livestreams/{slug}/chat/unpin-message` | Yes | Unpin message (owner/mod/admin) |

### Redis Storage Keys

| Key Pattern | Purpose | TTL |
|-------------|---------|-----|
| `livestream:chat:{livestreamId}` | Chat message list (last 100) | Until stream ends |
| `livestream:chat:pinned:{livestreamId}` | Pinned message | 24 hours |
| `livestream:chat:settings:{livestreamId}` | Chat mode settings | Until stream ends |
| `livestream:chat:mute:{livestreamId}:{userId}` | Individual user mute | Configurable minutes |
| `livestream:chat:muted_users:{livestreamId}` | Set of all muted user IDs | Until stream ends |
| `livestream:chat:slow:{livestreamId}:{userId}` | Slow mode rate limit | Configurable seconds |

### Message Types (broadcast payload `type` field)

| Type | Description |
|------|-------------|
| `chat` | Regular chat message |
| `chat_delete` | Message deleted by mod/owner |
| `chat_pin` | Message pinned to top |
| `chat_unpin` | Pinned message removed |
| `chat_clear` | All chat messages cleared |
| `chat_settings` | Chat settings updated |
| `system` | System announcement message |
| `mute_status` | User mute/unmute notification |
| `moderator_status` | User gained/lost moderator status |
| `follow` | New follower event |
| `subscription` | New subscription event |
| `mp_tip` | Tip/donation event |

### Chat Message Payload

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "chat",
  "message": "Hello everyone!",
  "deleted": false,
  "ts": "2026-01-01T12:00:00.000Z",
  "reply_to": {
    "id": "previous-msg-uuid",
    "user": { "name": "UserA", "id": 42 }
  },
  "client_message_id": "client-generated-uuid",
  "sender_user_id": "123",
  "user": {
    "id": 123,
    "name": "Display Name",
    "user_name": "username",
    "badges": ["owner", "subscriber"],
    "name_color": "#e84393",
    "is_verify": true,
    "badge": "gold"
  }
}
```

### User Badges

| Badge | Condition |
|-------|-----------|
| `owner` | User owns the livestream |
| `moderator` | User is assigned as moderator |
| `subscriber` | User has active subscription to streamer |

### Name Colors

- **Default:** Deterministic HSL color generated from user ID hash
- **Custom:** Set via `POST /api/user/chat-color` (brightness validated)

### Chat Commands

| Command | Syntax | Permission | Description |
|---------|--------|------------|-------------|
| `/clear` | `/clear` | Owner, Moderator | Clears all chat messages |
| `/susturma` | `/susturma "username" minutes` | Moderator with `canMuteUser` | Mutes a user for N minutes (Turkish: "muting") |

### Moderation Settings

Retrieved via `GET /api/livestreams/{slug}/chat/state`, updated via `POST /api/livestreams/{slug}/chat/settings`:

```json
{
  "chatSettings": {
    "enabled": true,
    "slowMode": 2,
    "emojiOnly": false,
    "followersOnly": false,
    "subscribersOnly": false
  }
}
```

| Setting | Type | Range | Description |
|---------|------|-------|-------------|
| `enabled` | boolean | - | Enable/disable chat entirely |
| `slowMode` | integer | 0-600 | Seconds between messages per user |
| `emojiOnly` | boolean | - | Only emoji messages allowed |
| `followersOnly` | boolean | - | Only followers can chat |
| `subscribersOnly` | boolean | - | Only subscribers can chat |

---

## 2. DM/Group Chat

### Key Files

| Component | File Path |
|-----------|-----------|
| Chat Controller | `packages/shaunsocial/chat/src/Http/Controllers/Api/ChatController.php` |
| Voice Controller | `packages/shaunsocial/chat/src/Http/Controllers/Api/ChatVoiceController.php` |
| ChatRoom Model | `packages/shaunsocial/chat/src/Models/ChatRoom.php` |
| ChatMessage Model | `packages/shaunsocial/chat/src/Models/ChatMessage.php` |
| ChatRoomMember Model | `packages/shaunsocial/chat/src/Models/ChatRoomMember.php` |
| ChatMessageItem Model | `packages/shaunsocial/chat/src/Models/ChatMessageItem.php` |
| MessageSentEvent | `packages/shaunsocial/chat/src/Events/MessageSentEvent.php` |
| Vue Components | `resources/js/pages/chat/` (ChatRoomDetail.vue, ChatRoomsList.vue, etc.) |

### WebSocket Channel

- **Channel:** `Social.Chat.Models.ChatRoom.{id}` (private — authorization required)
- **Authorization:** Checks `$room->canView($user->id)` in `routes/channels.php`

### API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/chat/get_request_count` | Yes | Count pending chat requests |
| GET | `/api/chat/get_room/{page?}` | Yes | List chat rooms (paginated) |
| GET | `/api/chat/get_room_request/{page?}` | Yes | List chat requests |
| GET | `/api/chat/search_room/{text}` | Yes | Search rooms by name |
| GET | `/api/chat/detail/{id}` | Yes | Get room details |
| GET | `/api/chat/get_room_message/{id}/{page?}` | Yes | Get room messages (paginated) |
| POST | `/api/chat/store_room` | Yes | Create new chat room |
| POST | `/api/chat/store_room_message` | Yes | Send message |
| POST | `/api/chat/store_room_seen/{id}` | Yes | Mark room as read |
| POST | `/api/chat/store_room_unseen/{id}` | Yes | Mark room as unread |
| POST | `/api/chat/store_room_status` | Yes | Update room status |
| POST | `/api/chat/store_room_notify` | Yes | Toggle room notifications |
| POST | `/api/chat/clear_room_message` | Yes | Clear room messages |
| POST | `/api/chat/upload_photo` | Yes | Upload photo to chat |
| POST | `/api/chat/upload_file` | Yes | Upload file to chat |
| POST | `/api/chat/store_audio` | Yes | Upload audio message |
| POST | `/api/chat/send_fund` | Yes | Send funds in chat |
| POST | `/api/chat/delete_message_item` | Yes | Delete message item |
| POST | `/api/chat/unsent_room_message/{id}` | Yes | Unsend a message |
| POST | `/api/chat/check_room_online` | Yes | Check if room members are online |
| POST | `/api/chat/store_active_room/{id}` | Yes | Mark room as active (typing indicator) |
| POST | `/api/chat/store_inactive_room/{id}` | Yes | Mark room as inactive |
| POST | `/api/chat/delete_room` | Yes | Delete a chat room |
| POST | `/api/chat/rooms/{room}/voice/join` | Yes | Join voice call in room |

### Broadcasting Events

| Event Class | Channel | broadcastAs() | Description |
|-------------|---------|---------------|-------------|
| `MessageSentEvent` | `Social.Core.Models.User.{userId}` (Private) | `Chat.MessageSentEvent` | New message sent |
| `MessageUnsentEvent` | Per-user private channel | `Chat.MessageUnsentEvent` | Message unsent/deleted |
| `RoomSeenEvent` | Per-user private channel | `Chat.RoomSeenEvent` | Read receipts |
| `RoomSeenSelfEvent` | Per-user private channel | `Chat.RoomSeenSelfEvent` | Self-read state sync |
| `RoomAcceptEvent` | Per-user private channel | `Chat.RoomAcceptEvent` | Room join/accept |
| `RoomUnreadEvent` | Per-user private channel | `Chat.RoomUnreadEvent` | Unread notification |

### Message Types

| Type | Description |
|------|-------------|
| `text` | Plain text message |
| `photo` | Photo attachment(s) |
| `file` | File attachment |
| `link` | Shared link with preview |
| `send_fund` | Fund transfer between users |

### ChatRoom Model Fields

| Field | Type | Description |
|-------|------|-------------|
| `code` | string | Unique room identifier |
| `name` | string | Room name |
| `is_group` | boolean | Group chat vs 1:1 DM |
| `last_message_id` | integer | FK to last message |

### ChatMessage Model Fields

| Field | Type | Description |
|-------|------|-------------|
| `user_id` | integer | Message author |
| `type` | string | Message type (text/photo/link/file/send_fund) |
| `content` | string | Message body (translatable) |
| `room_id` | integer | Parent chat room |
| `is_delete` | boolean | Soft-delete flag |
| `parent_message_id` | integer | For threaded replies |

---

## Real-time Architecture

### Broadcasting Stack

```
[Server Event] → Laravel Broadcasting → Pusher/Reverb → WebSocket → [Client Echo/Pusher-js]
```

### Configuration

- **Driver:** Configurable via `BROADCAST_DRIVER` env (supports: `pusher`, `ably`, `redis`, `reverb`, `null`)
- **Pusher Config:** `config/broadcasting.php` — `PUSHER_APP_KEY`, `PUSHER_APP_SECRET`, `PUSHER_APP_ID`
- **Reverb Config:** `config/reverb.php` — Laravel native WebSocket server at port 6060
- **Client:** Laravel Echo + `pusher-js` in frontend
