# Tech Stack

## Runtime & Framework

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Runtime | [Bun](https://bun.sh) | latest | JavaScript runtime, hot reload in dev |
| Framework | [Hono](https://hono.dev) | ^4.12.9 | Web framework with edge-first design |
| OpenAPI | @hono/zod-openapi | ^1.2.4 | Type-safe route definitions with Zod validation |
| API Docs | @scalar/hono-api-reference | ^0.10.5 | Interactive API documentation UI |
| Validation | [Zod](https://zod.dev) | ^4.3.6 | Schema validation |
| Language | TypeScript | ~5.9.3 | Strict mode, ESNext target |

## Database

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Database | PostgreSQL | — | Primary data store |
| ORM | [Drizzle ORM](https://orm.drizzle.team) | ^0.45.2 | Type-safe SQL query builder |
| Migrations | Drizzle Kit | ^0.31.10 | Schema migration tool |
| Driver | postgres | ^3.4.8 | PostgreSQL wire protocol client |
| Schema Bridge | drizzle-zod | ^0.8.3 | Auto-generate Zod schemas from Drizzle tables |

## Authentication

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Auth Provider | [better-auth](https://better-auth.com) | ^1.5.6 | Phone OTP, Google OAuth, email/password, sessions, admin roles |
| OTP Delivery | MSG91 | — | SMS OTP delivery for Indian phone numbers |

## AI & Content Processing

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| AI | Google Generative AI (Gemini 2.5 Flash Lite) | ^0.24.1 | PDF question extraction |
| PDF | pdf-lib | ^1.17.1 | PDF parsing and image extraction |
| Image | @napi-rs/canvas | ^0.1.99 | Server-side image manipulation/cropping |

## Storage & Infrastructure

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| File Storage | Cloudflare R2 (via AWS S3 SDK) | ^3.1032.0 | Images, PDFs |
| Logging | Pino + hono-pino | ^10.3.1 | Structured JSON logging |
| ID Generation | nanoid | ^5.1.9 | Unique IDs for uploads |

## Dev Tooling

| Tool | Purpose |
|------|---------|
| pnpm ^10.33.0 | Package manager (monorepo via pnpm-workspace.yaml) |
| vite-plus | Formatting and linting |
| @faker-js/faker ^10.4.0 | Fake data generation (for future tests) |
| @types/bun | Bun type definitions |

## Utilities

| Library | Version | Purpose |
|---------|---------|---------|
| stoker | ^2.0.1 | Hono utilities (status codes, middleware, OpenAPI helpers) |

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  Clients                         │
│         (iOS App, Android App, Admin Web)        │
└──────────────────────┬──────────────────────────┘
                       │ HTTPS
                       ▼
┌─────────────────────────────────────────────────┐
│            Hono API Server (Bun)                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │  Routes   │ │ Middleware│ │   OpenAPI/Scalar │ │
│  │ (Zod-     │ │ (Auth,   │ │   API Docs       │ │
│  │  Validated│ │  Logger) │ │                  │ │
│  └────┬─────┘ └──────────┘ └──────────────────┘ │
│       │                                         │
│  ┌────▼──────────────────────────────────────┐  │
│  │          Business Logic Layer              │  │
│  │  Assessment Engine · Grading · Scheduler   │  │
│  │  AI Import · Coin Management (planned)     │  │
│  └────┬──────────────────────────────────────┘  │
└───────┼─────────────────────────────────────────┘
        │
   ┌────┼────────────┐
   ▼    ▼            ▼
┌──────┐ ┌──────┐ ┌──────────────┐
│Postgr│ │Cloud │ │ Google Gemini│
│eSQL  │ │flare │ │    AI        │
│      │ │  R2  │ │              │
└──────┘ └──────┘ └──────────────┘
```

## Live API

- **Production**: `https://api.pine.education`
- **API Reference UI**: `https://api.pine.education/api/reference`
- **OpenAPI Spec**: `https://api.pine.education/api/doc`
