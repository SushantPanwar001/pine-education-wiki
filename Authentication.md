# User Registration & Authentication

## Overview

Authentication is powered by [better-auth](https://better-auth.com) with two registration methods:

1. **Phone + OTP** — user enters phone number, receives OTP, sets password, account is created and phone-verified in one step.
2. **OAuth (Google) + Phone + OTP** — user signs in via Google OAuth, then must complete phone verification before accessing any protected routes.

Only phone-verified users can access the app. The server enforces this gate via `sessionMiddleware` which returns `403 PHONE_VERIFICATION_REQUIRED` for unverified users.

## Provider: better-auth v1.5.6

- **Configuration**: `src/lib/auth.ts`
- **Database tables**: `user`, `session`, `account`, `verification` (in `src/db/schema/auth.ts`)
- **Plugins**: `phoneNumber()`, `admin()`, `bearer()`
- **Admin roles**: `["admin"]`, default role: `"user"`

## Registration Methods

### 1. Phone Number + OTP

**OTP Provider**: MSG91 (`src/lib/msg91.ts`)
**Endpoints**: `POST /api/auth/sign-up/phone-number` (send OTP), `POST /api/auth/sign-in/phone-number` (verify + login)

**Flow**:

```
User                    Client                  Backend              MSG91
 │                        │                        │                  │
 │  Enter phone number    │                        │                  │
 │───────────────────────►│                        │                  │
 │                        │  POST /api/auth/       │                  │
 │                        │  sign-up/phone-number  │                  │
 │                        │  { phoneNumber }       │                  │
 │                        │───────────────────────►│                  │
 │                        │                        │                  │
 │                        │                        │  Validate phone  │
 │                        │                        │  (+91[6-9]XXX..) │
 │                        │                        │                  │
 │                        │                        │  Send OTP        │
 │                        │                        │─────────────────►│
 │                        │                        │                  │
 │                        │                        │  OTP sent        │
 │                        │                        │◄─────────────────│
 │                        │                        │                  │
 │                        │  { success: true }     │                  │
 │                        │◄───────────────────────│                  │
 │                        │                        │                  │
 │  Enter OTP + password  │                        │                  │
 │───────────────────────►│                        │                  │
 │                        │  POST /api/auth/       │                  │
 │                        │  sign-in/phone-number  │                  │
 │                        │  { phoneNumber,        │                  │
 │                        │    password, otp }     │                  │
 │                        │───────────────────────►│                  │
 │                        │                        │                  │
 │                        │                        │  Verify OTP      │
 │                        │                        │  Set phoneNumber  │
 │                        │                        │  Set phoneVerified│
 │                        │                        │  Link credential  │
 │                        │                        │  Create session   │
 │                        │                        │                  │
 │                        │  { user, token }       │                  │
 │                        │◄───────────────────────│                  │
 │                        │                        │                  │
 │  ── User has full access (phoneVerified: true)  │
```

**Validation Rules**:
- Phone number: Indian format only — `+91` followed by `[6-9]` and 9 more digits
- OTP: 4 digits, expires in 180 seconds
- Password: minimum and maximum length enforced (validated then hashed by better-auth)
- On first verification, a temporary email `{phone}@phone.temp` is generated (better-auth requires an email field)
- `phoneNumberVerified` is set to `true` immediately upon verification
- Subsequent logins use the same phone number with password + OTP

**Profile Fields** (collected post-registration):
- `name` — user's display name
- `grade` — class/grade (smallint)
- `stream` — e.g., "Science", "Commerce" (varchar 20)
- `state` — Indian state (varchar 50)

### 2. Google OAuth + Phone Verification

**Configuration**: `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` env vars
**Endpoints**:
- `POST /api/auth/sign-in/social` — Google OAuth (better-auth built-in)
- `POST /api/auth/verify-phone/send` — send OTP to phone (session required, phone unverified OK)
- `POST /api/auth/verify-phone/verify` — verify OTP and mark phone verified (session required, phone unverified OK)

**Flow**:

```
User              Client              Backend            Google        MSG91
 │                  │                    │                  │            │
 │  "Sign in with   │                    │                  │            │
 │   Google"        │                    │                  │            │
 │─────────────────►│                    │                  │            │
 │                  │  Google Sign-In    │                  │            │
 │                  │  SDK               │                  │            │
 │                  │──────────────────────────────────────►│            │
 │                  │                    │                  │            │
 │                  │  Auth code         │                  │            │
 │                  │◄──────────────────────────────────────│            │
 │                  │                    │                  │            │
 │                  │  POST /api/auth/   │                  │            │
 │                  │  sign-in/social    │                  │            │
 │                  │  { providerId:     │                  │            │
 │                  │    "google",       │                  │            │
 │                  │    code }          │                  │            │
 │                  │───────────────────►│                  │            │
 │                  │                    │  Exchange code   │            │
 │                  │                    │─────────────────►│            │
 │                  │                    │                  │            │
 │                  │                    │  User profile    │            │
 │                  │                    │◄─────────────────│            │
 │                  │                    │                  │            │
 │                  │                    │  Create account  │            │
 │                  │                    │  (phoneVerified: │            │
 │                  │                    │   false/null)    │            │
 │                  │                    │  Create session  │            │
 │                  │                    │                  │            │
 │                  │  { user, token }   │                  │            │
 │                  │◄───────────────────│                  │            │
 │                  │                    │                  │            │
 │                  │  ── user.phoneVerified is false ──    │            │
 │                  │  ── Client shows phone verification   │            │
 │                  │     modal ──                         │            │
 │                  │                    │                  │            │
 │  Enter phone     │                    │                  │            │
 │─────────────────►│                    │                  │            │
 │                  │  POST /api/auth/   │                  │            │
 │                  │  verify-phone/send │                  │            │
 │                  │  { phoneNumber }   │                  │            │
 │                  │───────────────────►│                  │            │
 │                  │                    │                  │            │
 │                  │                    │  Validate phone  │            │
 │                  │                    │  Send OTP        │───────────►│
 │                  │                    │                  │            │
 │                  │                    │  OTP sent        │◄───────────│
 │                  │                    │                  │            │
 │                  │  { success }       │                  │            │
 │                  │◄───────────────────│                  │            │
 │                  │                    │                  │            │
 │  Enter OTP       │                    │                  │            │
 │─────────────────►│                    │                  │            │
 │                  │  POST /api/auth/   │                  │            │
 │                  │  verify-phone/     │                  │            │
 │                  │  verify            │                  │            │
 │                  │  { phoneNumber,    │                  │            │
 │                  │    otp }           │                  │            │
 │                  │───────────────────►│                  │            │
 │                  │                    │                  │            │
 │                  │                    │  Verify OTP      │            │
 │                  │                    │  Set phoneNumber  │            │
 │                  │                    │  Set phoneVerified│            │
 │                  │                    │                  │            │
 │                  │  { verified: true }│                  │            │
 │                  │◄───────────────────│                  │            │
 │                  │                    │                  │            │
 │  ── User has full access (phoneVerified: true) ──       │            │
```

**Important**:
- The OAuth user gets a session token immediately after Google login, but `phoneNumberVerified` is `false`
- Any request to a protected route returns `403 { error: "PHONE_VERIFICATION_REQUIRED" }`
- The verify-phone endpoints use `sessionOnlyMiddleware` (session check without phone verification gate)
- Once verified, the user can access all protected routes normally

## Phone Verification Gate

The server enforces phone verification at the middleware level. Every protected route passes through `sessionMiddleware` which checks both session validity AND phone verification status.

**Error response when unverified**:
```json
{
  "error": "PHONE_VERIFICATION_REQUIRED",
  "message": "Phone number verification is required to access this resource"
}
```

**Client behavior**: On receiving `403` with `PHONE_VERIFICATION_REQUIRED`, the client should show the phone verification modal (phone input → OTP input → verified).

## Session Management

- **Mechanism**: Bearer token in `Authorization` header
- **Plugin**: `bearer()` from better-auth
- **Token format**: Better-auth managed session token
- **Session table**: Tracks session ID, user ID, expiry, IP, user agent

## Middleware

### Middleware Stack (`src/middlewares/auth.ts`)

| Middleware | Session | Phone Verified | Admin | Used By |
|-----------|---------|----------------|-------|---------|
| `sessionOnlyMiddleware` | Required | Not checked | No | `verify-phone/*` endpoints |
| `sessionMiddleware` | Required | Required | No | Assessment, coin routes |
| `adminMiddleware` | Required | Required | Yes | Admin, import routes |

**Route-level auth**:
| Route Group | Auth Level |
|-------------|-----------|
| `/api/auth/*` | None (public auth endpoints) |
| `/api/auth/verify-phone/*` | Session required, phone NOT required |
| `/api/exam` (GET) | None |
| `/api/question` | None |
| `/api/assessment/*` | Session + phone verified |
| `/api/import/*` | Admin + phone verified |
| `/api/admin/*` | Admin + phone verified |

## User Schema

| Field | Type | Description |
|-------|------|-------------|
| id | text PK | better-auth generated |
| name | text NOT NULL | Display name |
| email | text NOT NULL UNIQUE | Email or `{phone}@phone.temp` |
| emailVerified | boolean | Email verification status |
| image | text | Avatar URL |
| phoneNumber | varchar(255) UNIQUE | Indian mobile number |
| phoneNumberVerified | boolean | OTP verification status — **hard gate for app access** |
| grade | smallint | Class/grade |
| stream | varchar(20) | Study stream |
| state | varchar(50) | Indian state |
| role | text DEFAULT "user" | "user" or "admin" |
| banned | boolean DEFAULT false | Ban status |
| banReason | text | Reason for ban |
| banExpires | timestamp | Ban expiry |
| createdAt | timestamp | Registration date |
| updatedAt | timestamp | Last update |

## Mobile Integration Notes

### Phone + OTP Flow

1. App presents phone number input → calls `POST /api/auth/sign-up/phone-number`
2. App presents OTP + password input → calls `POST /api/auth/sign-in/phone-number`
3. Backend returns session token with `phoneVerified: true`
4. App stores token in Keychain (iOS) / EncryptedSharedPreferences (Android)
5. User has full access immediately

### Google OAuth Flow

1. Use native Google Sign-In SDK to obtain auth code
2. Send auth code to backend via `/api/auth/sign-in/social`
3. Backend returns session token — but `phoneVerified` is `false`
4. App checks `phoneVerified` flag and shows phone verification modal
5. User enters phone → `POST /api/auth/verify-phone/send`
6. User enters OTP → `POST /api/auth/verify-phone/verify`
7. Backend confirms verification → user has full access

### Handling 403 PHONE_VERIFICATION_REQUIRED

On any API call, if the client receives:
```json
{ "error": "PHONE_VERIFICATION_REQUIRED" }
```
The client should:
1. Show the phone verification modal
2. Do NOT clear the session token — it's still valid
3. After successful verification, retry the original request

### Token Storage

- iOS: Keychain (`kSecClassGenericPassword`)
- Android: Encrypted SharedPreferences (AndroidX Security)
- Never store tokens in `UserDefaults` (iOS) or `SharedPreferences` (Android)

### Session Refresh

- better-auth handles session renewal automatically
- If session expires, client receives `401` → redirect to login
- Phone verification status persists across sessions (stored on user record, not session)
