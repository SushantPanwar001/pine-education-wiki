# Deployment

## Infrastructure

| Component | Provider | Details |
|-----------|----------|---------|
| API Server | **Vultr VPS** | Mumbai region |
| Database | **Supabase** | Managed PostgreSQL with built-in backups, monitoring, and connection pooling |
| Deployment Tool | **Coolify** | Self-hosted PaaS with GitHub App integration |
| File Storage | **Cloudflare R2** | S3-compatible object storage |
| OTP Service | **MSG91** | SMS delivery for Indian numbers |
| AI Service | **Google Gemini** | 2.5 Flash Lite for PDF extraction |
| Domain | `api.pine.education` | Production API endpoint |

## Deployment Flow

```
GitHub Push          Coolify              Vultr VPS
    │                   │                    │
    │  Push to main     │                    │
    │──────────────────►│                    │
    │                   │  Build & Deploy    │
    │                   │───────────────────►│
    │                   │                    │
    │                   │  Health Check      │
    │                   │───────────────────►│
    │                   │                    │
    │                   │  200 OK            │
    │                   │◄───────────────────│
    │                   │                    │
    │  Deploy Complete  │                    │
    │◄──────────────────│                    │
```

Coolify monitors the GitHub repository. On push to the configured branch (typically `main`), Coolify automatically:
1. Pulls the latest code
2. Builds the application (Bun runtime)
3. Deploys to the Vultr VPS
4. Runs health checks
5. Replaces the running instance with zero-downtime deployment

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | Supabase PostgreSQL connection string (use pooler URL for production: `postgresql://postgres.[project-ref]:6543/postgres`) |
| `BETTER_AUTH_SECRET` | Yes | Secret key for session signing |
| `BETTER_AUTH_URL` | Yes | Base URL of the API (`https://api.pine.education`) |
| `GOOGLE_CLIENT_ID` | Yes | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Yes | Google OAuth client secret |
| `MSG91_AUTH_KEY` | Yes | MSG91 API key for OTP delivery |
| `MSG91_TEMPLATE_ID` | Yes | MSG91 OTP template ID |
| `R2_ACCESS_KEY_ID` | Yes | Cloudflare R2 access key |
| `R2_SECRET_ACCESS_KEY` | Yes | Cloudflare R2 secret key |
| `R2_BUCKET_NAME` | Yes | Cloudflare R2 bucket name |
| `R2_ENDPOINT` | Yes | Cloudflare R2 endpoint URL |
| `R2_PUBLIC_URL` | Yes | Public URL for R2 objects |
| `GEMINI_API_KEY` | Yes | Google Gemini AI API key |
| `PORT` | No | Server port (default: 3000) |
| `NODE_ENV` | No | `production` / `development` |

## Local Development

```bash
# Install dependencies
pnpm install

# Run database migrations
pnpm --filter backend db:migrate

# Start dev server with hot reload
pnpm --filter backend dev
# or directly:
cd apps/backend && bun run --hot src/index.ts
```

## Monitoring & Maintenance

### Current State
- **Logging**: Pino structured logging to stdout (captured by Coolify)
- **Uptime monitoring**: Not yet configured
- **Error tracking**: Not yet configured
- **Backups**: Supabase managed — automatic daily backups with point-in-time recovery (verify project settings)
