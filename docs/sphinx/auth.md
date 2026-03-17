# Authentication System — M2Portal

> Based on actual repository code inspection. All references point to real files and implementations.

---

## Overview

M2Portal uses **Laravel Sanctum** for API token authentication. The system supports email/username/phone login, user registration, two-factor authentication, and password recovery.

---

## Key Files

| Component | File Path |
|-----------|-----------|
| Auth Controller | `packages/shaunsocial/core/src/Http/Controllers/Api/AuthController.php` |
| Auth Routes | `packages/shaunsocial/core/routes/api.php` |
| User Model | `packages/shaunsocial/core/src/Models/User.php` |
| Token Model | `packages/shaunsocial/core/src/Models/Sanctum/PersonalAccessToken.php` |
| Auth Config | `config/auth.php` |
| Sanctum Config | `config/sanctum.php` |
| API Middleware | `packages/shaunsocial/core/src/Http/Middleware/ApplicationApi.php` |
| Token Cookie Middleware | `packages/shaunsocial/core/src/Http/Middleware/ApplicationToken.php` |
| Login Validation | `packages/shaunsocial/core/src/Http/Requests/Auth/LoginValidate.php` |
| Signup Validation | `packages/shaunsocial/core/src/Http/Requests/Auth/SignupValidate.php` |
| 2FA Validation | `packages/shaunsocial/core/src/Http/Requests/Auth/LoginWithCodeValidate.php` |

---

## API Routes

### Authentication

| Method | Path | Handler | Auth | Description |
|--------|------|---------|------|-------------|
| POST | `/api/auth/login` | `AuthController@login` | No | Login with email/username/phone + password |
| POST | `/api/auth/signup` | `AuthController@signup` | No | Register new user |
| POST | `/api/auth/login_with_code` | `AuthController@login_with_code` | No | Complete 2FA login |
| GET | `/api/auth/signup/config` | `AuthController@config` | No | Get signup field requirements |
| GET | `/api/auth/logout` | `AuthController@logout` | Yes | Logout and invalidate token |

### Password Recovery

| Method | Path | Handler | Auth | Description |
|--------|------|---------|------|-------------|
| POST | `/api/user/send_code_forgot_password` | `UserController@send_code_forgot_password` | No | Send password reset code |
| POST | `/api/user/check_code_forgot_password` | `UserController@check_code_forgot_password` | No | Verify reset code |
| POST | `/api/user/store_forgot_password` | `UserController@store_forgot_password` | No | Set new password |
| POST | `/api/user/change_password` | `UserController@change_password` | Yes | Change password (logged in) |

### Two-Factor Authentication

| Method | Path | Handler | Auth | Description |
|--------|------|---------|------|-------------|
| GET | `/api/two_factor/get_current` | `TwoFactorController@get_current` | Yes | Get current 2FA status |
| POST | `/api/two_factor/remove` | `TwoFactorController@remove` | Yes | Remove 2FA |
| POST | `/api/two_factor/send_setup_email` | `TwoFactorController@send_setup_email` | Yes | Setup email-based 2FA |
| POST | `/api/two_factor/verify_setup_email` | `TwoFactorController@verify_setup_email` | Yes | Verify email 2FA setup |
| POST | `/api/two_factor/send_setup_phone` | `TwoFactorController@send_setup_phone` | Yes | Setup phone-based 2FA |
| POST | `/api/two_factor/verify_setup_phone` | `TwoFactorController@verify_setup_phone` | Yes | Verify phone 2FA setup |
| POST | `/api/two_factor/get_code_app` | `TwoFactorController@get_code_app` | Yes | Setup authenticator app |
| POST | `/api/two_factor/verify_code_app` | `TwoFactorController@verify_code_app` | Yes | Verify authenticator app |
| POST | `/api/two_factor/get_login_current` | `TwoFactorController@get_login_current` | No | Get 2FA method for login |
| POST | `/api/two_factor/send_login_code` | `TwoFactorController@send_login_code` | No | Send 2FA code during login |
| POST | `/api/two_factor/verify_login_code` | `TwoFactorController@verify_login_code` | No | Verify 2FA code during login |

---

## Authentication Flow

### Standard Login

```
Client                          Server
  |                                |
  |  POST /api/auth/login          |
  |  { email, password }           |
  |------------------------------->|
  |                                |  Validate credentials
  |                                |  Check user active status
  |                                |  If 2FA enabled → return error with two_factory_code
  |                                |  Else → createToken('authToken')
  |  { data: { user, token } }    |
  |<-------------------------------|
  |                                |
  |  Subsequent API requests       |
  |  Authorization: Bearer {token} |
  |------------------------------->|
```

### Two-Factor Login

```
Client                          Server
  |                                |
  |  POST /api/auth/login          |
  |  { email, password }           |
  |------------------------------->|
  |  Error: two_factor required    |
  |  { two_factory_code: "..." }   |
  |<-------------------------------|
  |                                |
  |  POST /api/auth/login_with_code|
  |  { code: "123456" }           |
  |------------------------------->|
  |  { data: { user, token } }    |
  |<-------------------------------|
```

### Signup

```
Client                          Server
  |                                |
  |  GET /api/auth/signup/config   |
  |------------------------------->|
  |  { birthdate, location,        |
  |    gender requirements }       |
  |<-------------------------------|
  |                                |
  |  POST /api/auth/signup         |
  |  { name, user_name, email,     |
  |    password, ... }             |
  |------------------------------->|
  |                                |  Validate fields per settings
  |                                |  Create user
  |                                |  Process referral code
  |                                |  createToken('authToken')
  |  { data: { user, token } }    |
  |<-------------------------------|
```

---

## Token Management

**Token Creation:**
```php
$user->createToken('authToken')->plainTextToken
// Returns format: "1|abcdef123456..."
```

**PersonalAccessToken Customizations:**
- `findToken()` — overrides default lookup with Redis/cache support (cached by SHA256 hash)
- `getTokenableAttribute()` — uses `findByField()` for cached user lookups
- Auto-sets `last_used_at` on creation
- Tracks `is_page` field for page account tokens
- Prunable: auto-deletes tokens not used for `config('shaun_core.core.auto_delete_day')` days
- On deletion: clears cache, updates user `has_active` status, syncs `UserPageToken`

**Token Expiration:** `null` (no automatic expiration; relies on pruning)

---

## Middleware Chain

All API requests pass through this middleware stack (in `ApplicationApi.php`):

1. **Offline Mode Check** — returns 503 if site is offline without valid access code
2. **Locale Setup** — sets locale from `Accept-Language` header
3. **Bearer Token Extraction** — extracts Sanctum user from `Authorization: Bearer` header
4. **User Active Check** — blocks inactive users with 403 (except `/logout`)
5. **Email Verification** — blocks unverified email if feature enabled (except whitelisted routes)
6. **Phone Verification** — blocks unverified phone if feature enabled (except whitelisted routes)
7. **Page Parent Validation** — ensures page accounts have a parent user

**`ApplicationToken` middleware:** Converts `access_token` cookie to `Authorization: Bearer` header (for web SPA requests).

---

## Guards Configuration

From `config/auth.php`:

```php
'guards' => [
    'web'   => ['driver' => 'session', 'provider' => 'users'],
    'admin' => ['driver' => 'session', 'provider' => 'users'],
],
'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model'  => Packages\ShaunSocial\Core\Models\User::class,
    ],
],
```

---

## Request Validation

### Login (`LoginValidate`)
- `email` — required, validated as email/username/phone, must exist in DB
- `password` — required string
- Optional reCAPTCHA spam check

### Signup (`SignupValidate`)
- `name` — required, max 64
- `user_name` — required, max 128, unique
- `email` — required, valid email, max 255, unique
- `password` — required, custom validation rules
- `birthday` — required if `feature.require_birth`
- `gender_id` — required if `feature.require_gender`
- `phone_number` — required if `feature.phone_verify`
- `country_id`, `state_id`, `city_id` — required if `feature.require_location`
- `ref_code` — required if `feature.invite_only`
- Optional reCAPTCHA spam check

---

## Security Features

| Feature | Implementation |
|---------|---------------|
| Token Authentication | Laravel Sanctum with Redis-cached token lookups |
| Rate Limiting | `throttle:login` on login endpoint |
| Two-Factor Auth | Email, Phone, Authenticator App (TOTP) |
| Email Verification | Enforced per setting before API access |
| Phone Verification | Enforced per setting before API access |
| reCAPTCHA | Optional on login/signup forms |
| IP Restrictions | Optional whitelist/blacklist via `restrictIpAddress` middleware |
| Password Reset | Code-based verification flow with email throttling |
| Token Pruning | Auto-deletion of inactive tokens after configurable days |
| User Status Check | Inactive users blocked from all API access except logout |
| CSRF Protection | Via Sanctum stateful authentication for SPA |
