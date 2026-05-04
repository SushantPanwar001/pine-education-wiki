# Features

## Implemented Features

### 1. Exam Management

**Status**: Live
**Endpoints**: `GET/POST /api/exam`, `GET/PATCH /api/exam/:id`
**Auth**: Session required

- Create exams with name, duration (minutes), total marks, timing mode (continuous/pausable), and navigation mode (linear/free)
- Exams contain multiple sections, each with configurable marking scheme, optional per-section timer, and question count
- Update exam properties and active status
- List all active exams with sections

### 2. Question Bank

**Status**: Live
**Endpoints**: `POST /api/question`, `POST /api/question/bulk`, `GET /api/question/exam/:examId`, `PATCH /api/question/:id`, `DELETE /api/question/:id`
**Auth**: Session required

- Three question types: **MCQ** (single correct), **MSQ** (multiple correct), **NAT** (numerical answer type)
- Question content in Markdown with LaTeX support
- Images embedded via Cloudflare R2 URLs
- Per-question custom marking schemes (correct, incorrect, partial marks)
- Configurable answer validation: choice selection, exact value, or range (min/max/precision)

### 3. AI-Powered PDF Import

**Status**: Live
**Endpoints**: `POST /api/import/upload`, `GET /api/import/batch/:id/status`, `GET /api/import/pending`, `GET /api/import/batch/:id`, `PATCH /api/import/question/:id/accept`, `DELETE /api/import/question/:id`, `PATCH /api/import/question/:id/skip`

- Upload exam PDFs for automatic question extraction via Google Gemini 2.5 Flash Lite
- PDF parsing extracts questions, options, answers, and images
- Admin review workflow: accept, reject, or skip individual extracted questions
- Batch processing with status tracking (processing/completed/failed)
- Questions start in `pending_review` status, move to `active` on acceptance

### 4. Assessment / CBT Engine

**Status**: Live
**Endpoints**: `POST /api/assessment`, `GET /api/assessment/:id`, `PATCH /api/assessment/:id/answer`, `POST /api/assessment/:id/heartbeat`, `GET /api/assessment/:id/palette`, `POST /api/assessment/:id/submit`

- Full computer-based test experience
- Question palette showing per-section question status
- Answer actions: save, clear, mark for review, save + mark for review
- Heartbeat-based timer synchronization (client sends periodic heartbeats)
- Server-side countdown timer with 30-second scheduler for auto-submit on expiry
- Two timing modes: continuous (single timer for entire exam) and pausable (per-section timers)
- Two navigation modes: linear (sequential sections) and free (jump between sections)

### 5. Grading Engine

**Status**: Live
**Location**: `src/lib/assessment-submit.ts`

- Automatic grading on submission or timer expiry
- MCQ: exact match for correct/incorrect
- MSQ: partial credit for partially correct selections
- NAT: range-based validation with configurable precision
- Per-question custom marking schemes override section defaults
- Score calculation with marks per correct, incorrect, and partial answers

### 6. Admin Dashboard & Analytics

**Status**: Live
**Endpoints**: `GET /api/admin/dashboard`, `GET /api/admin/users`, `GET /api/admin/users/:id`, `PATCH /api/admin/users/:id/role`, `GET /api/admin/assessments`, `GET /api/admin/assessments/:id`, `GET /api/admin/exams/:id/analytics`

- Dashboard with user counts, active exams, total assessments, average scores, 30-day trends
- User management: list (paginated, searchable), detail view, role assignment (admin/user)
- Assessment listing with filters (exam, status)
- Assessment detail with full response breakdown
- Exam analytics: per-question accuracy, attempt counts, score distribution

### 7. Image Upload

**Status**: Live
**Endpoint**: `POST /api/upload`
**Auth**: Session required

- Upload images to Cloudflare R2 via S3-compatible API
- Returns public URL for embedding in question content

### 8. Section Management

**Status**: Live
**Endpoint**: `PATCH /api/section/:id`
**Auth**: Session required

- Update section properties: name, marks, duration, question count, sort order

### 9. Phone Verification Gate (OAuth)

**Status**: Live
**Endpoints**: `POST /api/auth/verify-phone/send`, `POST /api/auth/verify-phone/verify`
**Auth**: Session required (phone verification NOT required)

- OAuth users must verify phone number before accessing protected routes
- Server-enforced gate: all protected routes return `403 PHONE_VERIFICATION_REQUIRED` for unverified users
- See [AUTH.md](./Authentication) for full flow details

---

## Planned Features

### 10. Pine Coin System

**Status**: Planned
**See**: [PINE_COIN.md](./Pine-Coin)

- Virtual currency for assessment purchases
- Configurable price per exam
- Bundle deals for coin purchase
- Transaction ledger
- Admin manual credit

### 11. Payment Gateway Integration

**Status**: Planned (deferred — awaiting App Store account verification)

- Razorpay for Indian users (UPI, cards, net banking)
- Stripe as fallback for international
- Apple/Google in-app purchases for mobile subscriptions
- Server-side webhook handling for payment confirmation

### 12. Ad-Based Coin Rewards

**Status**: Planned (deferred)

- Users earn Pine Coins by watching ads
- Server-side verification of ad completion (AdMob/Unity Ads)
- Configurable reward amounts

### 13. Store Compliance Endpoints

**Status**: Planned
**See**: [STORE_READINESS.md](./Store-Readiness)

- Privacy policy endpoint
- Terms of service endpoint
- Data deletion / account deletion endpoint
- App-attestation / device integrity verification

---

## API Endpoint Summary

All API endpoints require authentication. No public endpoints exist except auth endpoints (`/api/auth/*`). Every protected route enforces phone verification via `sessionMiddleware`.

| Category | Endpoints | Auth Level |
|----------|-----------|------------|
| Auth | 4 | None (public) / Session-only (verify-phone) |
| Exam | 4 | Session + phone verified |
| Section | 1 | Session + phone verified |
| Question | 5 | Session + phone verified |
| Assessment | 7 | Session + phone verified |
| Import | 7 | Admin |
| Upload | 1 | Session + phone verified |
| Admin | 7 | Admin |
| **Total** | **36** | |

Interactive API documentation is available at `GET /api/reference` (Scalar UI) and the OpenAPI spec at `GET /api/doc`.
