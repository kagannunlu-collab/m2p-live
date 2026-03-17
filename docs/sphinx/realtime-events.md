# Real-time Events — M2Portal

> Based on actual repository code inspection. All references point to real files and implementations.

---

## Overview

M2Portal uses **Laravel Broadcasting** with support for **Pusher**, **Laravel Reverb**, **Ably**, and **Redis** drivers. Real-time events are delivered via WebSocket connections using **Laravel Echo** on the client side.

**Configuration:**
- Broadcasting: `config/broadcasting.php` — driver set via `BROADCAST_DRIVER` env
- Reverb: `config/reverb.php` — native WebSocket server at port 6060
- Client: `laravel-echo` + `pusher-js`

---

## Channel Definitions

Defined in `routes/channels.php`:

| Channel | Type | Authorization | Purpose |
|---------|------|---------------|---------|
| `Social.Core.Models.User.{id}` | Private | `user->id === id` | Per-user notifications |
| `Social.Chat.Models.ChatRoom.{id}` | Private | `room->canView(user->id)` | Chat room messages |
| `portalk.guild.{guildId}` | Presence | Guild member check | Guild presence (online status) |
| `portalk.guild.{guildId}.events` | Private | Guild member check | Guild events (member join/leave, etc.) |
| `portalk.channel.{channelId}` | Private | `VIEW_CHANNELS` permission | Channel-specific events |
| `portalk.voice.{channelId}` | Private | `CONNECT` permission + voice channel | Voice channel presence |
| `portalk.user.{userId}` | Private | `user->id === userId` | Portalk user notifications |
| `livestream.chat.{slug}` | **Public** | No auth required | Livestream chat events |

---

## Livestream Events

### LivestreamChatBroadcast

**File:** `packages/shaunsocial/livestream/src/Events/LivestreamChatBroadcast.php`  
**Implements:** `ShouldBroadcastNow`  
**Channel:** `livestream.chat.{slug}` (Public)  
**Event Name:** `livestream.chat.event`

All livestream real-time communication flows through this single event with different `type` values:

#### Chat Message (`type: chat`)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "chat",
  "message": "Hello everyone!",
  "deleted": false,
  "ts": "2026-01-01T12:00:00.000Z",
  "reply_to": {
    "id": "prev-msg-uuid",
    "user": { "name": "UserA", "id": 42 }
  },
  "client_message_id": "client-uuid",
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

#### Chat Delete (`type: chat_delete`)

```json
{
  "id": "uuid",
  "type": "chat_delete",
  "target_id": "message-id-to-delete",
  "ts": "2026-01-01T12:00:00.000Z",
  "by": { "id": 123, "name": "Moderator Name" }
}
```

#### Chat Pin (`type: chat_pin`)

```json
{
  "id": "uuid",
  "type": "chat_pin",
  "message": { /* full message object */ },
  "ts": "2026-01-01T12:00:00.000Z",
  "by": { "id": 123, "name": "Moderator Name" }
}
```

#### Chat Unpin (`type: chat_unpin`)

```json
{
  "id": "uuid",
  "type": "chat_unpin",
  "ts": "2026-01-01T12:00:00.000Z",
  "by": { "id": 123, "name": "Moderator Name" }
}
```

#### Chat Clear (`type: chat_clear`)

```json
{
  "id": "uuid",
  "type": "chat_clear",
  "ts": "2026-01-01T12:00:00.000Z",
  "by": { "id": 123, "name": "Moderator Name" }
}
```

#### Chat Settings Updated (`type: chat_settings`)

```json
{
  "id": "uuid",
  "type": "chat_settings",
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
    "overlayBgGradient": "linear-gradient(...)",
    "overlayBorderColor": "#5e3f2a",
    "overlayDisplayDuration": 6,
    "chatTextColor": "#ffffff",
    "chatBgGradient": "linear-gradient(...)",
    "chatBorderColor": "rgba(255,255,255,0.1)",
    "soundFollow": "",
    "soundTip": "",
    "soundSubscribe": "",
    "soundFollowVolume": 0.5,
    "soundTipVolume": 0.5,
    "soundSubscribeVolume": 0.5
  },
  "ts": "2026-01-01T12:00:00.000Z"
}
```

#### Mute Status (`type: mute_status`)

```json
{
  "id": "uuid",
  "type": "mute_status",
  "user_id": 456,
  "muted": true,
  "remaining_seconds": 300,
  "ts": "2026-01-01T12:00:00.000Z"
}
```

#### Moderator Status (`type: moderator_status`)

```json
{
  "id": "uuid",
  "type": "moderator_status",
  "user_id": 456,
  "is_moderator": true,
  "permissions": {
    "canDeleteMessage": true,
    "canMuteUser": true
  },
  "ts": "2026-01-01T12:00:00.000Z"
}
```

#### System Message (`type: system`)

```json
{
  "id": "uuid",
  "type": "system",
  "message": "Username, artık bu kanalın moderatörüsün.",
  "ts": "2026-01-01T12:00:00.000Z",
  "by": { "id": 123, "name": "Owner Name" }
}
```

#### Follow Event (`type: follow`)

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

#### Subscription Event (`type: subscription`)

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

#### Tip Event (`type: mp_tip`)

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

## Chat Events (DM/Group)

### MessageSentEvent

**File:** `packages/shaunsocial/chat/src/Events/MessageSentEvent.php`  
**Implements:** `Base` (extends `ShouldBroadcastNow`)  
**Channel:** `Social.Core.Models.User.{userId}` (Private)  
**Event Name:** `Chat.MessageSentEvent`

Payload: Custom data array with message details.

### MessageUnsentEvent

**File:** `packages/shaunsocial/chat/src/Events/MessageUnsentEvent.php`  
**Channel:** Per-user private channel  
**Event Name:** `Chat.MessageUnsentEvent`

### RoomSeenEvent

**File:** `packages/shaunsocial/chat/src/Events/RoomSeenEvent.php`  
**Channel:** Per-user private channel  
**Event Name:** `Chat.RoomSeenEvent`

### RoomSeenSelfEvent

**File:** `packages/shaunsocial/chat/src/Events/RoomSeenSelfEvent.php`  
**Channel:** Per-user private channel  
**Event Name:** `Chat.RoomSeenSelfEvent`

### RoomAcceptEvent

**File:** `packages/shaunsocial/chat/src/Events/RoomAcceptEvent.php`  
**Channel:** Per-user private channel  
**Event Name:** `Chat.RoomAcceptEvent`

### RoomUnreadEvent

**File:** `packages/shaunsocial/chat/src/Events/RoomUnreadEvent.php`  
**Channel:** Per-user private channel  
**Event Name:** `Chat.RoomUnreadEvent`

---

## Core User Events

### UserPresenceStatusUpdated

**File:** `packages/shaunsocial/core/src/Events/UserPresenceStatusUpdated.php`  
**Implements:** `ShouldBroadcastNow`  
**Channel:** `Social.Core.Models.User.{watcherId}` (Private)  
**Event Name:** `User.PresenceStatusUpdated`

```json
{
  "user_id": 123,
  "status": "online"
}
```

### UserPrivacySettingUpdated

**File:** `packages/shaunsocial/core/src/Events/UserPrivacySettingUpdated.php`  
**Implements:** `ShouldBroadcastNow`  
**Channel:** `Social.Core.Models.User.{watcherId}` (Private)  
**Event Name:** `User.PrivacySettingUpdated`

```json
{
  "user_id": 123,
  "allow_chat_voice_call": true
}
```

---

## Discord/Portalk Events

**Directory:** `app/Events/Discord/`

| Event | Channel | broadcastAs() | Payload |
|-------|---------|---------------|---------|
| `MessageSent` | `portalk.channel.{channelId}` | `message.sent` | MessageResource data |
| `MessageEdited` | `portalk.channel.{channelId}` | `message.edited` | MessageResource data |
| `MessageDeleted` | `portalk.channel.{channelId}` | `message.deleted` | `{channel_id, message_id}` |
| `MessagePinned` | `portalk.channel.{channelId}` | `message.pinned` | `{channel_id, message_id, is_pinned}` |
| `UserTyping` | `portalk.channel.{channelId}` | `user.typing` | UserPublicResource + `is_typing` |
| `ReactionAdded` | `portalk.channel.{channelId}` | `reaction.added` | `{channel_id, message_id, emoji, user_id}` |
| `ReactionRemoved` | `portalk.channel.{channelId}` | `reaction.removed` | `{channel_id, message_id, emoji, user_id}` |
| `MemberJoined` | `portalk.guild.{guildId}.events` | `member.joined` | `{guild_id, user_id}` |
| `MemberLeft` | `portalk.guild.{guildId}.events` | `member.left` | `{guild_id, user_id}` |
| `PresenceUpdated` | `portalk.guild.{guildId}` (Presence) | `presence.updated` | `{guild_id, user_id, status, user_data}` |
| `VoiceStatusUpdated` | `portalk.guild.{guildId}.events` | `voice.status.updated` | `{guild_id, channel_id, user_id, is_muted, is_deafened, connected}` |
| `MembershipChanged` | `portalk.user.{userId}` | `membership.changed` | `{user_id, guild_id, action}` |
| `MentionNotification` | `portalk.user.{userId}` | `mention.received` | `{user_id, guild_id, channel_id, message_id, preview}` |
| `GuildChanged` | `portalk.guild.{guildId}.events` | `guild.{action}` | `{guild_id, action, guild_data}` |
| `ChannelChanged` | `portalk.guild.{guildId}.events` | `channel.{action}` | `{guild_id, channel_id, action, channel_data}` |
| `CategoryChanged` | `portalk.guild.{guildId}.events` | `category.{action}` | `{guild_id, category_id, action, category_data}` |
| `ModerationAction` | `portalk.guild.{guildId}.events` | `moderation.action` | `{guild_id, action, target_user_id, meta}` |

---

## Event Summary

| System | Events | Channel Type | Description |
|--------|--------|-------------|-------------|
| Livestream | 1 event, 12 types | Public | All chat/overlay/moderation via single event |
| DM/Group Chat | 6 events | Private (per-user) | Messages, read receipts, room management |
| Core User | 2 events | Private (per-user) | Presence and privacy updates |
| Discord/Portalk | 17 events | Private/Presence | Full Discord-like real-time communication |
| **Total** | **26 events** | | |
