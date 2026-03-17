# Data Models — M2Portal

> Based on actual repository code inspection. All references point to real files and implementations.

---

## Livestream Models

### Livestream

**File:** `packages/shaunsocial/livestream/src/Models/Livestream.php`

| Field | Type | Description |
|-------|------|-------------|
| `user_id` | integer | Owner user ID |
| `title` | string | Stream title (max 100 chars) |
| `slug` | string | URL-friendly slug (from username) |
| `thumbnail` | string | Stream thumbnail URL |
| `room_name` | string | LiveKit room name (`live_user_{id}`) |
| `is_live` | boolean | Stream is currently active |
| `viewer_count` | integer | Current cached viewer count |
| `is_promoted` | boolean | Stream is highlighted/promoted |
| `promoted_until` | datetime | Promotion expiration time |
| `ingress_id` | string | OME ingress identifier |
| `ingress_rtmp_url` | string | RTMP URL for OBS |
| `ingress_stream_key` | string | 32-char hex stream key |
| `stream_slug` | string | OME stream slug |
| `playback_url` | string | Full HLS/LLHLS playback URL |
| `provider` | string | Provider type (`ome_llhls`) |

**Relationships:**
- `user()` → `belongsTo(User::class, 'user_id')`

**Key Methods:**
- `ensureOmeSlugAndPlayback()` — creates/updates OME streaming URLs, generates stream key if missing

---

### LivestreamSession

**File:** `packages/shaunsocial/livestream/src/Models/LivestreamSession.php`

| Field | Type | Description |
|-------|------|-------------|
| `user_id` | integer | Session owner |
| `livestream_id` | integer | Parent Livestream FK |
| `title` | string | Session title |
| `started_at` | datetime | Session start time |
| `ended_at` | datetime | Session end time |
| `duration_seconds` | integer | Total duration in seconds |
| `peak_viewers` | integer | Peak concurrent viewers |
| `avg_viewers` | integer | Average viewer count |
| `unique_viewers` | integer | Total unique viewers |
| `unique_chatters` | integer | Users who sent chat messages |
| `new_followers` | integer | New followers during stream |
| `new_subscribers` | integer | New subscribers during stream |
| `tips_amount` | integer | Total tips in MP units |
| `tips_count` | integer | Number of tips received |
| `chat_messages` | integer | Total chat messages sent |

**Relationships:**
- `user()` → `belongsTo(User::class, 'user_id')`
- `livestream()` → `belongsTo(Livestream::class, 'livestream_id')`

---

## User Model

**File:** `packages/shaunsocial/core/src/Models/User.php`

### Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name |
| `email` | string | Email address |
| `password` | string | Hashed password |
| `user_name` | string | Username (unique) |
| `role_id` | integer | Role FK |
| `bio` | string | Short biography |
| `about` | string | Extended about text |
| `location` | string | Location text |
| `gender_id` | integer | Gender FK |
| `birthday` | date | Date of birth |
| `language` | string | Preferred language |
| `timezone` | string | User timezone |
| `country_id` | integer | Country FK |
| `state_id` | integer | State FK |
| `city_id` | integer | City FK |
| `phone_number` | string | Phone number |
| `is_active` | boolean | Account active status |
| `has_active` | boolean | Has active token |
| `email_verified` | boolean | Email verified flag |
| `phone_verified` | boolean | Phone verified flag |
| `is_page` | boolean | Is a page account |
| `verify_status` | enum | Verification status |
| `privacy` | string | Privacy settings |
| `chat_name_color` | string | Custom chat name color |
| `presence_status` | string | Online/offline status |
| `follower_count` | integer | Follower count cache |
| `following_count` | integer | Following count cache |
| `post_count` | integer | Post count cache |
| `subscriber_count` | integer | Subscriber count cache |
| `chat_count` | integer | Unread chat count |
| `notify_count` | integer | Notification count |
| `ref_code` | string | Referral code |
| `earn_amount` | integer | Total earnings |
| `earn_fee_amount` | integer | Total fees paid |
| `wallet_notify_lower` | boolean | Low balance notification |

### Key Methods

| Method | Description |
|--------|-------------|
| `isModerator()` | Check if user is moderator or super admin |
| `isSuperAdmin()` | Check if user is super admin |
| `hasPermission($permission)` | Check specific permission |
| `getFollow($userId)` | Get follow relationship |
| `canFollow($viewerId)` | Check if viewer can follow |
| `canSendMessage($viewerId)` | Check if viewer can send messages |
| `canViewProfile($viewerId)` | Check profile visibility |
| `canView($viewerId, $checkPrivacy)` | Comprehensive view check |
| `checkBlock($userId)` | Check if blocked by user |
| `addBlock($userId, $isPage)` | Block a user |
| `isVerify()` | Check verified status |

### Traits

`HasCacheQueryFields`, `IsSubject`, `HasStorageFiles`, `HasCover`, `HasReport`, `HasAvatar`, `IsSubjectNotification`, `HasLink`

---

## Chat Models

### ChatRoom

**File:** `packages/shaunsocial/chat/src/Models/ChatRoom.php`

| Field | Type | Description |
|-------|------|-------------|
| `code` | string | Unique room identifier |
| `name` | string | Room name |
| `is_group` | boolean | Group chat vs 1:1 DM |
| `last_message_id` | integer | FK to last message |

**Key Methods:** `setViewer()`, `getMembers()`, `canView()`, `canSendMessage()`, `isOnline()`, `getRoomTwoUser()`

### ChatMessage

**File:** `packages/shaunsocial/chat/src/Models/ChatMessage.php`

| Field | Type | Description |
|-------|------|-------------|
| `user_id` | integer | Author user ID |
| `type` | string | `text`, `photo`, `link`, `file`, `send_fund` |
| `content` | string | Message body (translatable) |
| `room_id` | integer | Parent room FK |
| `is_delete` | boolean | Soft-delete flag |
| `parent_message_id` | integer | Reply thread FK |

### ChatRoomMember

**File:** `packages/shaunsocial/chat/src/Models/ChatRoomMember.php`

Tracks membership with status (`accepted`, `pending`, etc.).

### ChatMessageItem

**File:** `packages/shaunsocial/chat/src/Models/ChatMessageItem.php`

Stores message attachments (photos, files, audio).

---

## Paid Content Models

### TipPackage

**File:** `packages/shaunsocial/paid_content/src/Models/TipPackage.php`

Defines available tip/donation amounts.

### UserSubscriber

**File:** `packages/shaunsocial/paid_content/src/Models/UserSubscriber.php`

Tracks active subscriptions between users.

### SubscriberPackage

**File:** `packages/shaunsocial/paid_content/src/Models/SubscriberPackage.php`

Defines subscription tier packages.

### UserPostPaid

**File:** `packages/shaunsocial/paid_content/src/Models/UserPostPaid.php`

Tracks paid post purchases.

---

## Wallet Models

**Package:** `packages/shaunsocial/wallet/src/Models/`

| Model | Description |
|-------|-------------|
| `WalletTransaction` | All wallet transactions (deposits, tips, purchases) |
| `WalletOrder` | Purchase/top-up orders |
| `WalletWithdraw` | Withdrawal requests |
| `WalletPackage` | Pre-defined wallet top-up packages |
| `WalletPaymentType` | Supported payment methods |
| `WalletTransactionType` | Transaction categories |
| `WalletTransactionSubType` | Transaction sub-categories |
| `WalletNotifyBalance` | Low balance notifications |

---

## Core Social Models

**Package:** `packages/shaunsocial/core/src/Models/`

| Model | Description |
|-------|-------------|
| `Post`, `PostItem`, `PostStatistic` | Post content with media items and analytics |
| `Comment`, `CommentItem`, `CommentReply` | Post comments and replies |
| `Like`, `Bookmark` | Content interactions |
| `UserFollow`, `UserBlock` | Social relationships |
| `Poll`, `PollItem`, `PollVote` | Polling system |
| `UserNotification` | Push/in-app notifications |
| `Hashtag`, `HashtagFollow`, `HashtagTrending` | Hashtag system |
| `Role`, `Permission`, `RolePermission` | RBAC |
| `Report`, `ReportCategory` | Content reporting |
| `TwoFactorProvider`, `UserTwoFactor` | Two-factor authentication |
| `StorageFile`, `StorageService` | File storage |
| `Audio`, `Video`, `Link` | Media types |
| `Setting`, `SettingGroup` | Application settings |
| `Theme`, `Language`, `Country`, `State`, `City` | Localization |
| `Page`, `Menu`, `MenuItem` | CMS |
| `Invite`, `InviteHistory` | Referral system |
| `Currency` | Multi-currency support |

---

## Group Models

**Package:** `packages/shaunsocial/group/src/Models/`

| Model | Description |
|-------|-------------|
| `Group` | Group entity |
| `GroupCategory` | Group categories |
| `GroupMember` | Group membership |
| `GroupMemberRequest` | Join requests |
| `GroupBlock` | Blocked users in groups |
| `GroupRule` | Group rules |
| `GroupStatistic` | Group analytics |

---

## Discord Models (Portalk)

**Location:** `app/Models/Discord/`

| Model | Description |
|-------|-------------|
| `Guild` | Discord-style guild/server |
| `Channel`, `ChannelCategory` | Text/voice channels |
| `Message`, `MessageAttachment`, `MessageReaction` | Messages and reactions |
| `Role`, `Permission`, `ChannelPermissionOverride` | Permission system |
| `GuildMember` | Guild membership |
| `Ban`, `ModerationLog` | Moderation |
| `ChannelReadState` | Read state tracking |
| `Invite` | Guild invites |

---

## Additional Package Models

| Package | Models | Description |
|---------|--------|-------------|
| `story` | `Story`, `StoryItem`, `StoryView`, `StorySong`, `StoryBackground` | Stories system |
| `advertising` | `Advertising`, `AdvertisingDelivery`, `AdvertisingStatistic` | Ad platform |
| `user_page` | `UserPageInfo`, `UserPageAdmin`, `UserPageCategory`, `UserPageToken` | Creator pages |
| `user_subscription` | `UserSubscription`, `UserSubscriptionPlan`, `UserSubscriptionPackage` | Subscription tiers |
| `user_verify` | `UserVerifyFile` | Verification documents |
| `chatbot` | `ChatbotHistory`, `ChatbotProvider` | AI chatbot |
| `ai_provider` | `AiProvider`, `AiProviderKey` | AI integration |
| `ai_features` | `AiFeatureTask`, `AiFeatureTaskItem` | AI features |
| `vibb` | `VibbSong`, `VibbPostSong` | Short-form video music |
| `gateway` | `Gateway`, `OrderBase` | Payment gateways |
