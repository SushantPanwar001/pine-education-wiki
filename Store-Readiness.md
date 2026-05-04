# Store Readiness

## Overview

This document covers two aspects of backend readiness for App Store (Apple) and Google Play Store submission:

1. **API Readiness Checklist** — technical production-readiness of the backend
2. **Store Review Compliance** — backend requirements imposed by Apple and Google review guidelines

---

## 1. API Readiness Checklist

### Security

| Item | Status | Notes |
|------|--------|-------|
| HTTPS enforced | **Done** | Production API at `api.pine.education` uses HTTPS |
| Input validation (Zod) | **Done** | All routes validated via `@hono/zod-openapi` |
| SQL injection prevention | **Done** | Drizzle ORM parameterized queries |
| CORS configuration | **Done** | Configured in `src/lib/create-app.ts` |
| Rate limiting | **Missing** | No rate limiter configured. Need `hono-rate-limiter` or similar |
| Request size limits | **Missing** | No explicit body size limits for file uploads |
| Bearer token auth | **Done** | All protected routes use `Authorization: Bearer` |
| Admin role enforcement | **Done** | `adminMiddleware` checks role |
| Ban system | **Done** | `banned`, `banReason`, `banExpires` fields on user |
| Environment variables | **Done** | Secrets in `.env`, not committed |
| Error message sanitization | **Partial** | Structured errors via Zod, but some handlers may leak internal details |

### Reliability

| Item | Status | Notes |
|------|--------|-------|
| Auto-submit on timer expiry | **Done** | 30-second scheduler in `src/lib/scheduler.ts` |
| Database transactions | **Done** | Assessment creation uses Drizzle transactions |
| Structured logging | **Done** | Pino with JSON output, pino-pretty in dev |
| Health check endpoint | **Missing** | No `/health` or `/ready` endpoint |
| Graceful shutdown | **Missing** | No signal handlers for SIGTERM/SIGINT |
| Connection pooling | **Review needed** | Verify postgres connection pool size and limits |

### Performance

| Item | Status | Notes |
|------|--------|-------|
| Database indexes | **Done** | Indexes on foreign keys, status fields, composite indexes on (userId, examId) |
| Pagination | **Partial** | Admin endpoints have pagination; user-facing endpoints may not |
| Caching | **Missing** | No caching layer for frequently accessed data (exam lists, etc.) |
| Response compression | **Missing** | No gzip/brotli middleware |

### Observability

| Item | Status | Notes |
|------|--------|-------|
| Request logging | **Done** | hono-pino logs all requests with duration |
| Error logging | **Done** | Global error handler logs errors |
| Monitoring / alerts | **Missing** | No uptime monitoring, no alerting |
| Metrics endpoint | **Missing** | No Prometheus metrics or similar |

### Action Items for Production

1. **Add rate limiting** — protect OTP endpoint (MSG91 costs money) and assessment creation
2. **Add health check** — `GET /health` returning database connectivity and uptime
3. **Add request size limits** — prevent abuse on upload endpoints
4. **Add graceful shutdown** — drain in-flight requests before exiting
5. **Add response compression** — gzip middleware for JSON responses
6. **Set up monitoring** — uptime checks on `api.pine.education`, alert on 5xx rates
7. **Review connection pooling** — ensure postgres pool handles peak load

---

## 2. Apple App Store Requirements

### Required Backend Endpoints

| Requirement | Endpoint Needed | Status |
|-------------|----------------|--------|
| Privacy policy | `GET /legal/privacy-policy` | **Missing** |
| Terms of service | `GET /legal/terms-of-service` | **Missing** |
| Data deletion (Apple requires account deletion capability) | `POST /api/user/delete-account` | **Missing** |
| App Transport Security (ATS) | N/A (HTTPS enforced) | **Done** |
| Sign in with Apple (if other social login is offered) | Configure in better-auth | **Missing** — required if Google Sign-In is offered |

### Apple Review Guidelines — Backend Considerations

1. **Sign in with Apple**: If the app offers Google Sign-In (or any third-party auth), Apple **requires** Sign in with Apple as an option. This means adding the Apple OAuth provider to better-auth.
2. **Account Deletion**: Apple requires apps that allow account creation to also offer account deletion. The backend must:
   - Delete or anonymize all user data (responses, assessments, coin transactions)
   - Revoke sessions
   - Return confirmation
3. **In-App Purchases**: If Pine Coins are purchased via the app (not just via web), Apple will require using their native IAP API with server-side receipt validation. **This is deferred per plan.**
4. **Data Privacy**: The privacy policy must disclose what data is collected, how it's used, and how users can request deletion.

---

## 3. Google Play Store Requirements

### Required Backend Endpoints

| Requirement | Endpoint Needed | Status |
|-------------|----------------|--------|
| Privacy policy | `GET /legal/privacy-policy` | **Missing** |
| Data deletion (Google requires account deletion + data deletion within official timeframes) | `POST /api/user/delete-account` | **Missing** |
| Google Play Billing (for in-app purchases) | Server-side purchase verification | **Missing — deferred** |

### Google Play Policies — Backend Considerations

1. **Data Deletion**: Google requires a data deletion mechanism. The same `POST /api/user/delete-account` endpoint satisfies both Apple and Google.
2. **Google Play Billing**: If Pine Coins are purchased via the Android app, Google requires using Google Play Billing with server-side purchase token verification. **Deferred per plan.**
3. **Content Rating**: The exam content must be appropriate for the target age group. No backend changes needed, but content moderation should be considered.
4. **App Content Declaration**: Google requires declaring app content (ads, purchases, user-generated content). Backend should support any required reporting.

---

## 4. Implementation Plan

### Phase 1 — Before Store Submission (Required)

| Task | Priority | Effort |
|------|----------|--------|
| Create `GET /legal/privacy-policy` endpoint | Critical | Low |
| Create `GET /legal/terms-of-service` endpoint | Critical | Low |
| Create `POST /api/user/delete-account` endpoint | Critical | Medium |
| Add rate limiting middleware | Critical | Low |
| Add `GET /health` endpoint | High | Low |
| Add request body size limits | High | Low |
| Review and sanitize all error responses | High | Medium |

### Phase 2 — Before Launch (Recommended)

| Task | Priority | Effort |
|------|----------|--------|
| Add Sign in with Apple (better-auth Apple provider) | High (if offering Google Sign-In) | Medium |
| Set up uptime monitoring (e.g., UptimeRobot, BetterStack) | High | Low |
| Add response compression (gzip) | Medium | Low |
| Add graceful shutdown handling | Medium | Low |
| Set up error alerting | Medium | Medium |

### Phase 3 — Post-Launch

| Task | Priority | Effort |
|------|----------|--------|
| Apple IAP server-side receipt validation | Depends on monetization | Medium |
| Google Play Billing purchase verification | Depends on monetization | Medium |
| Prometheus metrics endpoint | Low | Medium |
| Caching layer for exam metadata | Low | Low |

---

## 5. Account Deletion Endpoint Design

```
POST /api/user/delete-account
Authorization: Bearer <token>

Response 200:
{
  "message": "Account scheduled for deletion",
  "deletionDate": "2026-05-11T00:00:00Z"  // 7-day grace period
}
```

**Behavior**:
1. Immediately ban the user (prevent new logins)
2. Schedule data for deletion after a 7-day grace period (allows user to cancel)
3. On deletion:
   - Anonymize user record (replace PII with `[deleted]`)
   - Delete or anonymize all responses, assessments, coin transactions
   - Revoke all active sessions
   - Remove from any future analytics/aggregation
4. Store deletion record for compliance audit trail

**Grace period cancellation**:
```
POST /api/user/cancel-deletion
Authorization: Bearer <token>

Response 200:
{
  "message": "Account deletion cancelled"
}
```
