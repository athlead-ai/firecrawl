# Firecrawl Self-Hosted Deployment Plan v2 (Railway)

**Last Updated:** 2025-01-18
**For:** Athletic website scraping (coaches, rosters) from Neon database
**Stack:** Firecrawl (forked) + Railway + Neon PostgreSQL + Redis

---

## Overview

This deployment plan uses the **actual Firecrawl monorepo architecture** instead of custom implementations. Firecrawl already includes production-ready API, workers, and Playwright services.

**Key Changes from v1:**
- ✅ Use existing Firecrawl services (no custom worker code needed)
- ✅ Leverage monorepo structure (`apps/api`, `apps/playwright-service-ts`)
- ✅ Deploy to Railway with proper service separation
- ✅ Use NUQ (Neon-based queue) instead of Redis-only queuing
- ⚠️ No Fire-engine access (self-hosted limitation - may affect complex sites)

**Cost Estimate:** $30-60/month on Railway
- API Service: ~$10/month
- Worker Service: ~$15-25/month
- Playwright Service: ~$10-20/month
- Redis: ~$5/month (Railway)
- Neon PostgreSQL: Free tier or ~$10/month

---

## Phase 1: Local Setup & Testing

### 1.1 Prerequisites

✅ Already completed:
- Forked to `github.com/athlead-ai/firecrawl`
- Local clone at `/Users/coreyyoung/Desktop/Projects/firecrawl`
- Git remote updated

**Still need:**
- pnpm v9+ installed
- Docker Desktop (for local testing)
- Railway CLI: `npm install -g @railway/cli`

### 1.2 Environment Configuration

**Create `.env` file in `apps/api/`:**

```bash
cd apps/api
cp .env.example .env
```

**Required variables for Railway deployment:**

```bash
# ===== Core Services =====
NUM_WORKERS_PER_QUEUE=2
PORT=3002
HOST=0.0.0.0

# Railway Redis (your existing instance)
REDIS_URL=redis://maglev.proxy.rlwy.net:56930
REDIS_RATE_LIMIT_URL=redis://maglev.proxy.rlwy.net:56930

# Neon PostgreSQL for NUQ (persistent queue)
# Replace with your actual Neon connection string
NUQ_DATABASE_URL=postgresql://user:pass@ep-xxxx.us-east-2.aws.neon.tech/firecrawl?sslmode=require

# ===== Authentication =====
USE_DB_AUTHENTICATION=false  # Keep false for self-hosted simplicity

# ===== Optional but Recommended =====
# Bull Queue Admin Panel (change this!)
BULL_AUTH_KEY=your_secret_key_here

# OpenAI for AI extraction features (optional)
OPENAI_API_KEY=sk-...

# Playwright service (auto-configured in Railway)
PLAYWRIGHT_MICROSERVICE_URL=http://playwright-service:3000/scrape

# ===== Not Needed (Supabase) =====
# SUPABASE_ANON_TOKEN=
# SUPABASE_URL=
# SUPABASE_SERVICE_TOKEN=
```

### 1.3 Database Setup: NUQ Schema

**The NUQ schema is in `apps/nuq-postgres/nuq.sql`**

Run this SQL in your Neon database:

```bash
# Download the schema
cat apps/nuq-postgres/nuq.sql

# Option 1: Run via Neon dashboard SQL editor
# Copy/paste contents into Neon SQL editor and execute

# Option 2: Run via psql
psql "postgresql://user:pass@ep-xxxx.neon.tech/firecrawl?sslmode=require" -f apps/nuq-postgres/nuq.sql
```

**Verify tables created:**
```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public';
```

Expected tables:
- `nuq_jobs`
- `nuq_job_events`
- Other supporting tables

### 1.4 Local Testing with Docker

**Test with official Docker Compose first:**

```bash
# From repo root
docker compose build
docker compose up
```

This starts:
- API server (port 3002)
- Worker processes
- Redis (containerized)
- Playwright service (port 3000)

**Test the API:**
```bash
curl -X POST http://localhost:3002/v1/scrape \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://texastech.com/sports/football/roster/coaches",
    "formats": ["markdown"]
  }'
```

**Access Bull Queue UI:**
```
http://localhost:3002/admin/your_secret_key_here/queues
```

Once working locally, proceed to Railway deployment.

---

## Phase 2: Railway Deployment Architecture

### 2.1 Service Overview

Deploy **4 separate Railway services:**

```
┌─────────────────────────────────────────────────────┐
│  Railway Project: firecrawl-production              │
├─────────────────────────────────────────────────────┤
│                                                      │
│  1. firecrawl-api (Public)                          │
│     - Express API server                            │
│     - Routes: /v1/scrape, /v1/crawl, /v1/search    │
│     - Port: 3002                                    │
│     - Memory: 512MB                                 │
│                                                      │
│  2. firecrawl-worker (Private)                      │
│     - BullMQ job processors                         │
│     - Handles scraping/crawling jobs                │
│     - Memory: 2GB (for Playwright)                  │
│                                                      │
│  3. playwright-service (Private)                    │
│     - Headless browser automation                   │
│     - Port: 3000                                    │
│     - Memory: 1-2GB                                 │
│                                                      │
│  4. redis (Private)                                 │
│     - BullMQ queue backing                          │
│     - Already deployed ✅                           │
│                                                      │
│  External: Neon PostgreSQL                          │
│     - NUQ persistent queue                          │
│     - Job state management                          │
└─────────────────────────────────────────────────────┘
```

### 2.2 Dockerfile Strategy

**Important:** Firecrawl doesn't include Railway-specific Dockerfiles. We need to create them.

#### Service 1: API Server

**Create `Dockerfile.railway.api` in repo root:**

```dockerfile
FROM node:18-slim

# Install pnpm
RUN npm install -g pnpm@9

WORKDIR /app

# Copy workspace files
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY apps/api/package.json ./apps/api/
COPY apps/rust-sdk ./apps/rust-sdk

# Install dependencies
RUN pnpm install --frozen-lockfile

# Copy API source
COPY apps/api ./apps/api

# Build TypeScript
WORKDIR /app/apps/api
RUN pnpm build

# Expose port
EXPOSE 3002

# Start API server
CMD ["node", "dist/src/index.js"]
```

#### Service 2: Worker

**Create `Dockerfile.railway.worker` in repo root:**

```dockerfile
FROM node:18-slim

# Install Playwright dependencies
RUN apt-get update && apt-get install -y \
    fonts-liberation \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libcups2 \
    libdbus-1-3 \
    libdrm2 \
    libgbm1 \
    libgtk-3-0 \
    libnspr4 \
    libnss3 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    xdg-utils \
    && rm -rf /var/lib/apt/lists/*

# Install pnpm
RUN npm install -g pnpm@9

WORKDIR /app

# Copy workspace files
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY apps/api/package.json ./apps/api/
COPY apps/rust-sdk ./apps/rust-sdk

# Install dependencies
RUN pnpm install --frozen-lockfile

# Install Playwright browsers
RUN npx playwright install chromium

# Copy API source
COPY apps/api ./apps/api

# Build TypeScript
WORKDIR /app/apps/api
RUN pnpm build

# Start workers
CMD ["node", "dist/src/services/queue-worker.js"]
```

#### Service 3: Playwright Service

**Create `Dockerfile.railway.playwright` in repo root:**

```dockerfile
FROM node:18-slim

# Install Playwright dependencies
RUN apt-get update && apt-get install -y \
    fonts-liberation \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libcups2 \
    libdbus-1-3 \
    libdrm2 \
    libgbm1 \
    libgtk-3-0 \
    libnspr4 \
    libnss3 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    xdg-utils \
    && rm -rf /var/lib/apt/lists/*

# Install pnpm
RUN npm install -g pnpm@9

WORKDIR /app

# Copy workspace files
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY apps/playwright-service-ts/package.json ./apps/playwright-service-ts/

# Install dependencies
RUN pnpm install --frozen-lockfile

# Install Playwright browsers
RUN npx playwright install chromium

# Copy Playwright service source
COPY apps/playwright-service-ts ./apps/playwright-service-ts

# Build TypeScript
WORKDIR /app/apps/playwright-service-ts
RUN pnpm build

EXPOSE 3000

# Start Playwright service
CMD ["node", "dist/index.js"]
```

### 2.3 Railway Service Configuration

#### Deploy API Service

```bash
# In Railway dashboard:
# 1. New Service → GitHub Repo → athlead-ai/firecrawl
# 2. Service Name: firecrawl-api
# 3. Settings:
#    - Root Directory: /
#    - Dockerfile Path: Dockerfile.railway.api
#    - Public Domain: Enable (generates URL)
```

**Environment Variables (API):**
```
NUM_WORKERS_PER_QUEUE=2
PORT=3002
HOST=0.0.0.0
REDIS_URL=${{redis.REDIS_URL}}
REDIS_RATE_LIMIT_URL=${{redis.REDIS_URL}}
NUQ_DATABASE_URL=${{NEON_DATABASE_URL}}
USE_DB_AUTHENTICATION=false
BULL_AUTH_KEY=your_secret_key
PLAYWRIGHT_MICROSERVICE_URL=http://playwright-service.railway.internal:3000/scrape
OPENAI_API_KEY=${{OPENAI_API_KEY}}
```

**Health Check:**
```
Path: /health
Port: 3002
```

#### Deploy Worker Service

```bash
# In Railway dashboard:
# 1. New Service → GitHub Repo → athlead-ai/firecrawl
# 2. Service Name: firecrawl-worker
# 3. Settings:
#    - Root Directory: /
#    - Dockerfile Path: Dockerfile.railway.worker
#    - Public Domain: Disabled (internal only)
```

**Environment Variables (Worker):**
```
# Same as API, minus public-facing settings
NUM_WORKERS_PER_QUEUE=8
REDIS_URL=${{redis.REDIS_URL}}
REDIS_RATE_LIMIT_URL=${{redis.REDIS_URL}}
NUQ_DATABASE_URL=${{NEON_DATABASE_URL}}
USE_DB_AUTHENTICATION=false
PLAYWRIGHT_MICROSERVICE_URL=http://playwright-service.railway.internal:3000/scrape
OPENAI_API_KEY=${{OPENAI_API_KEY}}
```

**Resource Allocation:**
```
Memory: 2048 MB (for Playwright browser instances)
CPU: 2 vCPU
```

#### Deploy Playwright Service

```bash
# In Railway dashboard:
# 1. New Service → GitHub Repo → athlead-ai/firecrawl
# 2. Service Name: playwright-service
# 3. Settings:
#    - Root Directory: /
#    - Dockerfile Path: Dockerfile.railway.playwright
#    - Public Domain: Disabled
#    - Private Networking: Enable
```

**Environment Variables (Playwright):**
```
PORT=3000
```

**Resource Allocation:**
```
Memory: 2048 MB
CPU: 2 vCPU
```

---

## Phase 3: Athletic Site Scraping Integration

### 3.1 Use Firecrawl API (No Custom Code Needed)

**Your use case works with existing Firecrawl endpoints:**

```typescript
// Scrape coaches page
const response = await fetch('https://your-app.railway.app/v1/scrape', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    url: 'https://texastech.com/sports/football/roster/coaches',
    formats: ['markdown', 'html'],
    onlyMainContent: true,
  }),
});

const data = await response.json();
// data.data.markdown contains coach information
```

**For structured extraction (uses OpenAI):**

```typescript
const response = await fetch('https://your-app.railway.app/v1/scrape', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    url: 'https://texastech.com/sports/football/roster/coaches',
    formats: [
      {
        type: 'json',
        schema: {
          type: 'object',
          properties: {
            coaches: {
              type: 'array',
              items: {
                type: 'object',
                properties: {
                  name: { type: 'string' },
                  title: { type: 'string' },
                  email: { type: 'string' },
                  phone: { type: 'string' },
                },
              },
            },
          },
        },
      },
    ],
  }),
});

const data = await response.json();
// data.data.json.coaches contains structured coach data
```

### 3.2 Batch Scraping Script

**Create `scripts/batch-scrape.ts` in your application (not Firecrawl repo):**

```typescript
import { Pool } from 'pg';

const FIRECRAWL_URL = process.env.FIRECRAWL_URL || 'https://your-app.railway.app';
const pool = new Pool({ connectionString: process.env.NEON_DATABASE_URL });

interface School {
  id: number;
  name: string;
  sport: string;
  coach_url: string;
  roster_url: string;
}

async function scrapeSchool(school: School) {
  console.log(`Scraping ${school.name} - ${school.sport}...`);

  // Scrape coaches
  const coachResponse = await fetch(`${FIRECRAWL_URL}/v1/scrape`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      url: school.coach_url,
      formats: ['markdown'],
      onlyMainContent: true,
    }),
  });

  const coachData = await coachResponse.json();

  // Store in your database
  await pool.query(
    'INSERT INTO scraped_coaches (school_id, data, scraped_at) VALUES ($1, $2, NOW())',
    [school.id, coachData.data]
  );

  console.log(`✓ Scraped coaches for ${school.name}`);

  // Rate limit
  await new Promise(resolve => setTimeout(resolve, 2000));
}

async function main() {
  const { rows: schools } = await pool.query<School>(
    'SELECT * FROM schools WHERE scrape_enabled = true ORDER BY name'
  );

  console.log(`Found ${schools.length} schools to scrape`);

  for (const school of schools) {
    try {
      await scrapeSchool(school);
    } catch (error) {
      console.error(`Failed to scrape ${school.name}:`, error);
    }
  }

  await pool.end();
}

main();
```

### 3.3 Scheduled Scraping (Optional)

**Use Railway Cron Jobs or external scheduler:**

1. **Railway Cron (if available in your plan):**
   - Add cron service in Railway dashboard
   - Schedule: `0 2 * * 0` (Sunday 2 AM)
   - Command: `node scripts/batch-scrape.js`

2. **GitHub Actions (free alternative):**

```yaml
# .github/workflows/scrape.yml
name: Scheduled Scraping
on:
  schedule:
    - cron: '0 2 * * 0'  # Sunday 2 AM
  workflow_dispatch:  # Manual trigger

jobs:
  scrape:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
      - run: npm install
      - run: node scripts/batch-scrape.js
        env:
          FIRECRAWL_URL: ${{ secrets.FIRECRAWL_URL }}
          NEON_DATABASE_URL: ${{ secrets.NEON_DATABASE_URL }}
```

---

## Phase 4: Monitoring & Optimization

### 4.1 Bull Queue Dashboard

Access at: `https://your-app.railway.app/admin/your_secret_key/queues`

Monitor:
- Waiting jobs
- Active jobs
- Completed jobs
- Failed jobs

### 4.2 Resource Optimization

**Start conservatively:**
- API: 512MB RAM
- Worker: 2GB RAM (can scale to 4GB if needed)
- Playwright: 2GB RAM
- Monitor Railway metrics for 1 week

**Scaling triggers:**
- Queue depth > 50 jobs: Add worker replica
- Memory usage > 80%: Increase RAM
- CPU > 80%: Add vCPU or replicas

### 4.3 Cost Tracking

**Expected Monthly Cost:**

| Service | Resources | Cost |
|---------|-----------|------|
| API | 512MB, 0.5 vCPU | ~$10 |
| Worker (1x) | 2GB, 2 vCPU | ~$20 |
| Playwright | 2GB, 2 vCPU | ~$20 |
| Redis | 512MB | ~$5 |
| **Total** | | **~$55/month** |

**Optimizations:**
- Use Railway's usage-based pricing efficiently
- Scale workers down during off-peak hours
- Cache frequently scraped pages (7-day TTL)
- Batch similar jobs together

---

## Phase 5: Deployment Checklist

### Pre-Deployment

- [ ] Neon database created with NUQ schema
- [ ] Railway Redis deployed and connection string obtained
- [ ] `.env` file configured in `apps/api/`
- [ ] Local testing with `docker compose up` successful
- [ ] Dockerfiles created in repo root
- [ ] Changes committed to `athlead-ai/firecrawl` fork

### Deployment Steps

1. **Push Dockerfiles to GitHub:**
```bash
git add Dockerfile.railway.*
git commit -m "Add Railway deployment Dockerfiles"
git push origin main
```

2. **Deploy services in Railway (in order):**
   - [ ] Redis (already done)
   - [ ] Playwright Service (internal)
   - [ ] API Service (public)
   - [ ] Worker Service (internal)

3. **Configure environment variables** in each service

4. **Test deployment:**
```bash
curl https://your-app.railway.app/health
```

5. **Run test scrape:**
```bash
curl -X POST https://your-app.railway.app/v1/scrape \
  -H 'Content-Type: application/json' \
  -d '{"url": "https://firecrawl.dev", "formats": ["markdown"]}'
```

### Post-Deployment

- [ ] Monitor Railway logs for errors
- [ ] Check Bull Queue dashboard
- [ ] Run test batch scrape (5-10 schools)
- [ ] Monitor resource usage for 24 hours
- [ ] Adjust worker count if needed
- [ ] Set up monitoring/alerting (optional)

---

## Limitations & Workarounds

### No Fire-Engine Access

**What this means:**
- Limited anti-bot bypass capabilities
- May struggle with heavily protected sites
- No automatic proxy rotation

**Workarounds:**
1. Use Playwright for JavaScript-rendered sites (included)
2. Add custom headers/user agents in scrape requests
3. Implement rate limiting (built-in)
4. Use proxy service if needed (configure in .env)

### Athletic Site Compatibility

**Most athletic sites use:**
- Sidearm Sports (common) - Works with Playwright ✅
- PrestoSports - Works with basic scraping ✅
- Custom WordPress - Usually works ✅
- Heavy JavaScript sites - Use Playwright ✅

**Test your top 10 schools first** to verify compatibility.

---

## Troubleshooting

### Worker not processing jobs

```bash
# Check Railway logs
railway logs --service firecrawl-worker

# Check Redis connection
redis-cli -u $REDIS_URL ping

# Check NUQ database
psql $NUQ_DATABASE_URL -c "SELECT COUNT(*) FROM nuq_jobs WHERE state='waiting';"
```

### Playwright fails to launch

- Increase worker memory to 4GB
- Check Railway logs for browser launch errors
- Verify Playwright dependencies in Dockerfile

### High failure rate

- Check Bull Queue failed jobs
- Review error messages in Railway logs
- Test problematic URLs locally
- Adjust selectors or extraction methods

---

## Next Steps

1. **Complete Phase 1 local testing**
2. **Create Dockerfiles** (copy from this plan)
3. **Deploy to Railway** (start with API + Playwright)
4. **Test with 5-10 schools**
5. **Add workers as needed**
6. **Build batch scraping integration**
7. **Monitor and optimize costs**

**Estimated Timeline:**
- Week 1: Local setup + Railway deployment (API + Playwright)
- Week 2: Worker deployment + testing (10-20 schools)
- Week 3: Batch scraping + optimization
- Week 4: Production rollout (all schools)

---

## Resources

- **Firecrawl Official Docs:** https://docs.firecrawl.dev
- **Railway Docs:** https://docs.railway.app
- **Bull Queue:** https://docs.bullmq.io
- **Playwright:** https://playwright.dev

**Support:**
- Railway Discord: https://discord.gg/railway
- Firecrawl Discord: https://discord.gg/gSmWdAkdwd
