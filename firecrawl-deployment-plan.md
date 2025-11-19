# Complete Self-Hosted Firecrawl Deployment Plan

## Overview

This document provides a comprehensive plan for deploying a self-hosted Firecrawl instance optimized for scraping college athletic websites. The system will extract coach contact information and roster data from hundreds of college athletic departments.

**Technology Stack:**
- Firecrawl (forked and customized)
- BullMQ (job queuing)
- Redis (queue backing store)
- Playwright (browser automation)
- PostgreSQL (data storage)
- Railway (hosting platform)

**Expected Costs:**
- Self-hosted: ~$20-50/month
- vs. Firecrawl API: ~$200-500/month

---

## Phase 1: Environment Setup & Infrastructure

### 1.1 Fork and Clone Repository

```bash
# Fork firecrawl/firecrawl on GitHub to your account
git clone https://github.com/YOUR_USERNAME/firecrawl.git
cd firecrawl
```

### 1.2 Railway Project Setup

Create a new Railway project with these services:

**Service 1: Redis**
- Use Railway's Redis template
- Note the connection URL for later

**Service 2: PostgreSQL** (optional but recommended)
- For storing scrape history, analytics, caching results
- Use Railway's PostgreSQL template
- You already have Neon, so you could use that instead

### 1.3 Local Development Environment

```bash
# Install dependencies
npm install

# Copy environment template
cp .env.example .env
```

---

## Phase 2: Configuration

### 2.1 Environment Variables

Configure your `.env` file:

```bash
# Redis Configuration
REDIS_URL=redis://your-railway-redis-url:6379

# API Configuration
PORT=3002
HOST=0.0.0.0
API_URL=https://your-app.railway.app

# Worker Configuration
NUM_WORKERS=2  # Start with 2, scale based on load

# Playwright Configuration
PLAYWRIGHT_BROWSER=chromium
HEADLESS=true

# Rate Limiting
RATE_LIMIT_ENABLED=true
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW=60000  # 1 minute

# Optional: Proxy Configuration (for avoiding IP blocks)
# PROXY_SERVER=http://your-proxy:port
# PROXY_USERNAME=username
# PROXY_PASSWORD=password

# Scraping Configuration
MAX_CRAWL_DEPTH=3
TIMEOUT=30000  # 30 seconds per page

# Database (if using PostgreSQL for persistence)
DATABASE_URL=postgresql://user:pass@host:5432/firecrawl
```

### 2.2 Customize for Athletic Websites

Create custom extraction patterns in `src/scrapers/`:

```typescript
// src/scrapers/athletic-sites.ts
export const athleticSitePatterns = {
  sidearm: {
    coachSelectors: [
      '.sidearm-roster-player-name',
      '.coach-card .name',
      // Add Sidearm-specific selectors
    ],
    contactSelectors: [
      '.email-link',
      '.phone-number',
      // etc
    ]
  },
  prestosports: {
    // PrestoSports patterns
  },
  // Add other CMS patterns
};
```

---

## Phase 3: Railway Deployment Architecture

### 3.1 Service Structure

Deploy **three separate services** on Railway:

#### Service A: API Server

**Purpose**: Handles HTTP requests, adds jobs to BullMQ

```dockerfile
# Dockerfile.api
FROM node:18-slim

# Install Playwright browsers
RUN npx playwright install-deps chromium

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Expose API port
EXPOSE 3002

# Start API server only
CMD ["npm", "run", "start:api"]
```

**Railway Configuration**:
- Public domain enabled
- Environment variables from .env
- Connect to shared Redis service

#### Service B: Worker Service

**Purpose**: Processes scraping jobs from BullMQ queue

```dockerfile
# Dockerfile.worker
FROM node:18-slim

# Install Playwright and Chromium
RUN npx playwright install-deps chromium
RUN npx playwright install chromium

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Start worker processes
CMD ["npm", "run", "start:worker"]
```

**Railway Configuration**:
- No public domain needed
- Same environment variables
- Connect to shared Redis service
- **Scale**: Start with 1 instance, can add more

#### Service C: Redis (Already Setup)

- Your existing Redis instance
- Ensure it has enough memory (start with 512MB-1GB)

### 3.2 Package.json Scripts

Add these scripts to `package.json`:

```json
{
  "scripts": {
    "start:api": "node dist/api/server.js",
    "start:worker": "node dist/workers/worker.js",
    "start:all": "concurrently \"npm run start:api\" \"npm run start:worker\"",
    "build": "tsc",
    "dev": "ts-node-dev src/api/server.ts",
    "dev:worker": "ts-node-dev src/workers/worker.ts"
  }
}
```

---

## Phase 4: Code Modifications

### 4.1 API Server Setup

```typescript
// src/api/server.ts
import express from 'express';
import { Queue } from 'bullmq';
import Redis from 'ioredis';

const app = express();
app.use(express.json());

// Connect to Redis
const redis = new Redis(process.env.REDIS_URL);

// Create BullMQ queue
const scrapeQueue = new Queue('scraping', {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
  },
});

// Scrape endpoint
app.post('/api/scrape', async (req, res) => {
  const { url, extractType } = req.body;
  
  // Add job to queue
  const job = await scrapeQueue.add('scrape', {
    url,
    extractType, // 'coaches' or 'roster'
    timestamp: Date.now(),
  });

  res.json({
    jobId: job.id,
    status: 'queued',
  });
});

// Status endpoint
app.get('/api/status/:jobId', async (req, res) => {
  const job = await scrapeQueue.getJob(req.params.jobId);
  
  if (!job) {
    return res.status(404).json({ error: 'Job not found' });
  }

  const state = await job.getState();
  const progress = job.progress;
  
  res.json({
    jobId: job.id,
    state,
    progress,
    result: state === 'completed' ? job.returnvalue : null,
  });
});

const PORT = process.env.PORT || 3002;
app.listen(PORT, () => {
  console.log(`API server running on port ${PORT}`);
});
```

### 4.2 Worker Setup

```typescript
// src/workers/worker.ts
import { Worker } from 'bullmq';
import { chromium } from 'playwright';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

const worker = new Worker(
  'scraping',
  async (job) => {
    const { url, extractType } = job.data;
    
    console.log(`Processing job ${job.id}: ${url}`);
    
    // Launch browser
    const browser = await chromium.launch({
      headless: true,
      args: ['--no-sandbox', '--disable-setuid-sandbox'],
    });
    
    const page = await browser.newPage();
    
    try {
      // Navigate to URL
      await page.goto(url, { 
        waitUntil: 'networkidle',
        timeout: 30000 
      });
      
      // Extract data based on type
      let data;
      if (extractType === 'coaches') {
        data = await extractCoaches(page, url);
      } else if (extractType === 'roster') {
        data = await extractRoster(page, url);
      }
      
      await browser.close();
      
      // Update progress
      await job.updateProgress(100);
      
      return {
        success: true,
        url,
        data,
        extractedAt: new Date().toISOString(),
      };
      
    } catch (error) {
      await browser.close();
      throw error;
    }
  },
  {
    connection: redis,
    concurrency: 5, // Process 5 jobs simultaneously per worker
  }
);

// Extraction functions
async function extractCoaches(page, url) {
  // Detect CMS platform
  const html = await page.content();
  const isSidearm = html.includes('sidearm');
  const isPrestoSports = html.includes('prestosports');
  
  if (isSidearm) {
    return extractSidearmCoaches(page);
  } else if (isPrestoSports) {
    return extractPrestoCoaches(page);
  } else {
    return extractGenericCoaches(page);
  }
}

async function extractSidearmCoaches(page) {
  return await page.evaluate(() => {
    const coaches = [];
    // Sidearm-specific selectors
    document.querySelectorAll('.coach-card').forEach(card => {
      coaches.push({
        name: card.querySelector('.name')?.textContent?.trim(),
        title: card.querySelector('.title')?.textContent?.trim(),
        email: card.querySelector('a[href^="mailto:"]')?.href?.replace('mailto:', ''),
        phone: card.querySelector('.phone')?.textContent?.trim(),
      });
    });
    return coaches;
  });
}

async function extractPrestoCoaches(page) {
  return await page.evaluate(() => {
    const coaches = [];
    // PrestoSports-specific selectors
    document.querySelectorAll('.staff-member').forEach(member => {
      coaches.push({
        name: member.querySelector('.name')?.textContent?.trim(),
        title: member.querySelector('.title')?.textContent?.trim(),
        email: member.querySelector('a[href^="mailto:"]')?.href?.replace('mailto:', ''),
        phone: member.querySelector('.phone')?.textContent?.trim(),
      });
    });
    return coaches;
  });
}

async function extractGenericCoaches(page) {
  // Fallback generic extraction
  return await page.evaluate(() => {
    const coaches = [];
    const emails = Array.from(document.querySelectorAll('a[href^="mailto:"]'));
    
    emails.forEach(emailLink => {
      const parent = emailLink.closest('div, li, tr');
      if (parent) {
        coaches.push({
          name: parent.querySelector('h3, h4, .name, strong')?.textContent?.trim(),
          email: emailLink.href.replace('mailto:', ''),
          phone: parent.textContent.match(/\d{3}[-.]?\d{3}[-.]?\d{4}/)?.[0],
        });
      }
    });
    
    return coaches;
  });
}

async function extractRoster(page, url) {
  const html = await page.content();
  const isSidearm = html.includes('sidearm');
  
  if (isSidearm) {
    return extractSidearmRoster(page);
  } else {
    return extractGenericRoster(page);
  }
}

async function extractSidearmRoster(page) {
  return await page.evaluate(() => {
    const players = [];
    document.querySelectorAll('.sidearm-roster-player').forEach(player => {
      players.push({
        name: player.querySelector('.sidearm-roster-player-name')?.textContent?.trim(),
        position: player.querySelector('.sidearm-roster-player-position')?.textContent?.trim(),
        year: player.querySelector('.sidearm-roster-player-academic-year')?.textContent?.trim(),
        height: player.querySelector('.sidearm-roster-player-height')?.textContent?.trim(),
        weight: player.querySelector('.sidearm-roster-player-weight')?.textContent?.trim(),
        hometown: player.querySelector('.sidearm-roster-player-hometown')?.textContent?.trim(),
      });
    });
    return players;
  });
}

async function extractGenericRoster(page) {
  return await page.evaluate(() => {
    const players = [];
    document.querySelectorAll('table tr, .roster-player').forEach(row => {
      const cells = Array.from(row.querySelectorAll('td, .player-info'));
      if (cells.length >= 3) {
        players.push({
          name: cells[0]?.textContent?.trim(),
          position: cells[1]?.textContent?.trim(),
          year: cells[2]?.textContent?.trim(),
          height: cells[3]?.textContent?.trim(),
          weight: cells[4]?.textContent?.trim(),
        });
      }
    });
    return players;
  });
}

worker.on('completed', (job) => {
  console.log(`Job ${job.id} completed successfully`);
});

worker.on('failed', (job, err) => {
  console.error(`Job ${job?.id} failed:`, err);
});

console.log('Worker started and waiting for jobs...');
```

---

## Phase 5: Database Integration

### 5.1 Store Results in PostgreSQL

```sql
-- src/db/schema.sql
CREATE TABLE scrape_jobs (
  id SERIAL PRIMARY KEY,
  job_id VARCHAR(255) UNIQUE NOT NULL,
  url TEXT NOT NULL,
  extract_type VARCHAR(50) NOT NULL,
  status VARCHAR(50) NOT NULL,
  result JSONB,
  error TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
);

CREATE TABLE extracted_coaches (
  id SERIAL PRIMARY KEY,
  school_name VARCHAR(255),
  sport VARCHAR(100),
  name VARCHAR(255),
  title VARCHAR(255),
  email VARCHAR(255),
  phone VARCHAR(50),
  bio TEXT,
  social_media JSONB,
  scraped_at TIMESTAMP DEFAULT NOW(),
  source_url TEXT
);

CREATE TABLE extracted_rosters (
  id SERIAL PRIMARY KEY,
  school_name VARCHAR(255),
  sport VARCHAR(100),
  player_name VARCHAR(255),
  position VARCHAR(100),
  year VARCHAR(50),
  height VARCHAR(50),
  weight VARCHAR(50),
  hometown VARCHAR(255),
  social_media JSONB,
  scraped_at TIMESTAMP DEFAULT NOW(),
  source_url TEXT
);

-- Add indexes
CREATE INDEX idx_coaches_school_sport ON extracted_coaches(school_name, sport);
CREATE INDEX idx_rosters_school_sport ON extracted_rosters(school_name, sport);
CREATE INDEX idx_scrape_jobs_status ON scrape_jobs(status);
CREATE INDEX idx_coaches_email ON extracted_coaches(email);
```

### 5.2 Save Results After Extraction

```typescript
// src/db/saveResults.ts
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

export async function saveCoaches(coaches, metadata) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    for (const coach of coaches) {
      await client.query(
        `INSERT INTO extracted_coaches 
         (school_name, sport, name, title, email, phone, bio, social_media, source_url)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
         ON CONFLICT (email) DO UPDATE SET
         name = EXCLUDED.name,
         title = EXCLUDED.title,
         scraped_at = NOW()`,
        [
          metadata.schoolName,
          metadata.sport,
          coach.name,
          coach.title,
          coach.email,
          coach.phone,
          coach.bio,
          JSON.stringify(coach.socialMedia),
          metadata.sourceUrl,
        ]
      );
    }
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

export async function saveRoster(players, metadata) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    for (const player of players) {
      await client.query(
        `INSERT INTO extracted_rosters 
         (school_name, sport, player_name, position, year, height, weight, hometown, social_media, source_url)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)`,
        [
          metadata.schoolName,
          metadata.sport,
          player.name,
          player.position,
          player.year,
          player.height,
          player.weight,
          player.hometown,
          JSON.stringify(player.socialMedia),
          metadata.sourceUrl,
        ]
      );
    }
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

export async function saveJobStatus(jobId: string, status: string, result?: any, error?: string) {
  await pool.query(
    `INSERT INTO scrape_jobs (job_id, status, result, error, completed_at)
     VALUES ($1, $2, $3, $4, NOW())
     ON CONFLICT (job_id) DO UPDATE SET
     status = EXCLUDED.status,
     result = EXCLUDED.result,
     error = EXCLUDED.error,
     completed_at = EXCLUDED.completed_at`,
    [jobId, status, JSON.stringify(result), error]
  );
}
```

### 5.3 Update Worker to Save Results

```typescript
// Update worker.ts to save to database
import { saveCoaches, saveRoster, saveJobStatus } from './db/saveResults';

const worker = new Worker(
  'scraping',
  async (job) => {
    const { url, extractType, metadata } = job.data;
    
    try {
      // ... existing scraping code ...
      
      // Save to database
      if (extractType === 'coaches') {
        await saveCoaches(data, metadata);
      } else if (extractType === 'roster') {
        await saveRoster(data, metadata);
      }
      
      await saveJobStatus(job.id, 'completed', { url, data });
      
      return {
        success: true,
        url,
        data,
        extractedAt: new Date().toISOString(),
      };
      
    } catch (error) {
      await saveJobStatus(job.id, 'failed', null, error.message);
      throw error;
    }
  },
  { connection: redis, concurrency: 5 }
);
```

---

## Phase 6: Testing & Optimization

### 6.1 Local Testing

```bash
# Start Redis locally (or connect to Railway)
docker run -d -p 6379:6379 redis

# Start API in one terminal
npm run dev

# Start worker in another terminal
npm run dev:worker

# Test scraping
curl -X POST http://localhost:3002/api/scrape \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://texastech.com/sports/football/roster/coaches",
    "extractType": "coaches",
    "metadata": {
      "schoolName": "Texas Tech",
      "sport": "Football"
    }
  }'

# Check status
curl http://localhost:3002/api/status/JOB_ID
```

### 6.2 Load Testing

```bash
# Install artillery
npm install -g artillery

# Create test script (loadtest.yml)
```

```yaml
# loadtest.yml
config:
  target: "http://localhost:3002"
  phases:
    - duration: 60
      arrivalRate: 10
scenarios:
  - name: "Scrape coaches"
    flow:
      - post:
          url: "/api/scrape"
          json:
            url: "https://texastech.com/sports/football/roster/coaches"
            extractType: "coaches"
            metadata:
              schoolName: "Texas Tech"
              sport: "Football"
```

```bash
# Run load test
artillery run loadtest.yml
```

### 6.3 Performance Benchmarks

Test different website types:

```typescript
// scripts/benchmark.ts
const testUrls = [
  {
    name: 'Sidearm Sports',
    url: 'https://texastech.com/sports/football/roster/coaches',
    expectedTime: 10000, // 10 seconds
  },
  {
    name: 'PrestoSports',
    url: 'https://gobulldogs.com/sports/football/roster/coaches',
    expectedTime: 15000,
  },
  {
    name: 'Custom WordPress',
    url: 'https://example.edu/athletics/football/staff',
    expectedTime: 30000,
  },
];

async function benchmark() {
  for (const test of testUrls) {
    const start = Date.now();
    
    const response = await fetch('http://localhost:3002/api/scrape', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        url: test.url,
        extractType: 'coaches',
      }),
    });
    
    const { jobId } = await response.json();
    
    // Wait for completion
    let completed = false;
    while (!completed) {
      const status = await fetch(`http://localhost:3002/api/status/${jobId}`);
      const data = await status.json();
      
      if (data.state === 'completed' || data.state === 'failed') {
        completed = true;
        const elapsed = Date.now() - start;
        console.log(`${test.name}: ${elapsed}ms (expected: ${test.expectedTime}ms)`);
      }
      
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}

benchmark();
```

---

## Phase 7: Deployment to Railway

### 7.1 Deploy API Service

```bash
# In Railway dashboard:
# 1. Create new service "Firecrawl API"
# 2. Connect to your GitHub repo
# 3. Set root directory to "/"
# 4. Set Dockerfile path to "Dockerfile.api"
# 5. Add environment variables
# 6. Enable public domain
# 7. Deploy
```

**Environment Variables to Add:**
```
REDIS_URL=${REDIS_URL}
DATABASE_URL=${DATABASE_URL}
PORT=3002
NODE_ENV=production
PLAYWRIGHT_BROWSER=chromium
HEADLESS=true
```

### 7.2 Deploy Worker Service

```bash
# In Railway dashboard:
# 1. Create new service "Firecrawl Worker"
# 2. Connect to same GitHub repo
# 3. Set root directory to "/"
# 4. Set Dockerfile path to "Dockerfile.worker"
# 5. Add same environment variables
# 6. Do NOT enable public domain
# 7. Deploy
```

### 7.3 Monitor Deployment

```bash
# Check Railway logs
railway logs --service firecrawl-api
railway logs --service firecrawl-worker

# Monitor Redis
railway connect redis
redis-cli MONITOR

# Check queue stats
redis-cli
> LLEN bull:scraping:wait
> LLEN bull:scraping:active
> LLEN bull:scraping:failed
```

### 7.4 Scaling Workers

To handle more load:

```bash
# In Railway dashboard:
# 1. Go to Firecrawl Worker service
# 2. Click "Settings" > "Replicas"
# 3. Increase replica count (start with 2-3)
# 4. Monitor performance and adjust
```

**Resource Allocation:**
- API Service: 512MB RAM, 0.5 vCPU
- Worker Service (per replica): 2GB RAM, 1 vCPU
- Redis: 512MB-1GB RAM

---

## Phase 8: Create Batch Processing System

### 8.1 Batch Scraper Script

```typescript
// scripts/batchScrape.ts
import fetch from 'node-fetch';
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

interface School {
  name: string;
  sport: string;
  coachUrl: string;
  rosterUrl: string;
}

async function batchScrapeSchools(schools: School[]) {
  const API_URL = process.env.FIRECRAWL_API_URL || 'http://localhost:3002';
  
  const jobIds = [];
  
  for (const school of schools) {
    console.log(`Processing ${school.name} - ${school.sport}...`);
    
    // Scrape coaches
    const coachJob = await fetch(`${API_URL}/api/scrape`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        url: school.coachUrl,
        extractType: 'coaches',
        metadata: {
          schoolName: school.name,
          sport: school.sport,
        },
      }),
    });
    
    const coachResult = await coachJob.json();
    jobIds.push({ type: 'coaches', ...coachResult });
    
    // Scrape roster
    const rosterJob = await fetch(`${API_URL}/api/scrape`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        url: school.rosterUrl,
        extractType: 'roster',
        metadata: {
          schoolName: school.name,
          sport: school.sport,
        },
      }),
    });
    
    const rosterResult = await rosterJob.json();
    jobIds.push({ type: 'roster', ...rosterResult });
    
    console.log(`✓ Queued ${school.name} - ${school.sport}`);
    
    // Rate limiting: wait 1 second between schools
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  console.log(`\nQueued ${jobIds.length} total jobs`);
  return jobIds;
}

// Load schools from your database
async function loadSchools() {
  const { rows } = await pool.query(`
    SELECT 
      name, 
      sport, 
      coach_url as "coachUrl", 
      roster_url as "rosterUrl"
    FROM schools 
    WHERE scrape_enabled = true
    ORDER BY name, sport
  `);
  
  return rows;
}

// Monitor job progress
async function monitorJobs(jobIds: any[]) {
  const API_URL = process.env.FIRECRAWL_API_URL || 'http://localhost:3002';
  
  console.log('\nMonitoring job progress...\n');
  
  let completed = 0;
  let failed = 0;
  
  const checkInterval = setInterval(async () => {
    const statuses = await Promise.all(
      jobIds.map(async (job) => {
        const response = await fetch(`${API_URL}/api/status/${job.jobId}`);
        return response.json();
      })
    );
    
    completed = statuses.filter(s => s.state === 'completed').length;
    failed = statuses.filter(s => s.state === 'failed').length;
    const pending = statuses.filter(s => s.state === 'waiting' || s.state === 'active').length;
    
    console.log(`Progress: ${completed} completed, ${failed} failed, ${pending} pending`);
    
    if (completed + failed === jobIds.length) {
      clearInterval(checkInterval);
      console.log('\n✓ All jobs completed!');
      console.log(`Final: ${completed} successful, ${failed} failed`);
    }
  }, 5000);
}

// Run batch scrape
async function main() {
  console.log('Loading schools from database...\n');
  const schools = await loadSchools();
  
  console.log(`Found ${schools.length} schools to scrape\n`);
  console.log('Starting batch scrape...\n');
  
  const jobIds = await batchScrapeSchools(schools);
  
  // Optional: Monitor progress
  await monitorJobs(jobIds);
  
  await pool.end();
}

main().catch(console.error);
```

### 8.2 Scheduled Scraping with Cron

```typescript
// scripts/scheduledScrape.ts
import cron from 'node-cron';
import { batchScrapeSchools, loadSchools } from './batchScrape';

// Run every Sunday at 2 AM
cron.schedule('0 2 * * 0', async () => {
  console.log('Starting scheduled scrape...');
  
  const schools = await loadSchools();
  await batchScrapeSchools(schools);
  
  console.log('Scheduled scrape completed');
});

console.log('Cron job scheduled: Every Sunday at 2 AM');
```

### 8.3 Smart Incremental Updates

Only scrape schools that need updating:

```typescript
// scripts/incrementalScrape.ts
async function loadStaleSchools(daysOld = 7) {
  const { rows } = await pool.query(`
    SELECT DISTINCT
      s.name, 
      s.sport, 
      s.coach_url as "coachUrl", 
      s.roster_url as "rosterUrl"
    FROM schools s
    LEFT JOIN extracted_coaches c ON s.name = c.school_name AND s.sport = c.sport
    WHERE 
      s.scrape_enabled = true
      AND (
        c.scraped_at IS NULL 
        OR c.scraped_at < NOW() - INTERVAL '${daysOld} days'
      )
    ORDER BY c.scraped_at ASC NULLS FIRST
    LIMIT 100
  `);
  
  return rows;
}

async function main() {
  const staleSchools = await loadStaleSchools(7);
  console.log(`Found ${staleSchools.length} schools that need updating`);
  
  if (staleSchools.length > 0) {
    await batchScrapeSchools(staleSchools);
  }
}
```

---

## Phase 9: Cost Optimization

### 9.1 Implement Caching

```typescript
// src/cache/cache.ts
import { createHash } from 'crypto';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

function getCacheKey(url: string, extractType: string): string {
  return `cache:${createHash('md5').update(url + extractType).digest('hex')}`;
}

export async function getCachedResult(url: string, extractType: string) {
  const cacheKey = getCacheKey(url, extractType);
  const cached = await redis.get(cacheKey);
  
  if (cached) {
    console.log(`Cache hit for ${url}`);
    return JSON.parse(cached);
  }
  
  return null;
}

export async function setCachedResult(url: string, extractType: string, data: any) {
  const cacheKey = getCacheKey(url, extractType);
  // Cache for 7 days
  await redis.setex(cacheKey, 60 * 60 * 24 * 7, JSON.stringify(data));
  console.log(`Cached result for ${url}`);
}

// Update worker to check cache first
const worker = new Worker(
  'scraping',
  async (job) => {
    const { url, extractType, metadata } = job.data;
    
    // Check cache first
    const cached = await getCachedResult(url, extractType);
    if (cached) {
      return {
        success: true,
        url,
        data: cached,
        fromCache: true,
      };
    }
    
    // ... proceed with scraping if not cached ...
    
    // Save to cache after successful scrape
    await setCachedResult(url, extractType, data);
    
    return result;
  }
);
```

### 9.2 Proxy Rotation (Optional)

For high-volume scraping:

```typescript
// src/proxy/proxyManager.ts
const proxies = [
  'http://proxy1.example.com:8080',
  'http://proxy2.example.com:8080',
  'http://proxy3.example.com:8080',
];

let currentProxyIndex = 0;

export function getNextProxy() {
  const proxy = proxies[currentProxyIndex];
  currentProxyIndex = (currentProxyIndex + 1) % proxies.length;
  return proxy;
}

// Use in worker
const browser = await chromium.launch({
  headless: true,
  proxy: {
    server: getNextProxy(),
  },
});
```

### 9.3 Railway Resource Optimization

**Recommended Configuration:**

| Service | RAM | vCPU | Estimated Cost |
|---------|-----|------|----------------|
| API | 512MB | 0.5 | $5/month |
| Worker (x1) | 2GB | 1 | $10/month |
| Worker (x2) | 2GB | 1 | $10/month |
| Redis | 512MB | - | $5/month |
| **Total** | | | **$20-30/month** |

**Scaling Strategy:**
1. Start with 1 worker
2. Monitor queue depth
3. Add workers if queue consistently > 50 jobs
4. Each worker can handle ~100 jobs/hour

### 9.4 Cost Comparison

**Self-Hosted vs Firecrawl API:**

For 500 schools × 2 URLs each = 1,000 scrapes/week:

| Solution | Monthly Cost | Notes |
|----------|--------------|-------|
| Firecrawl API | $200-500 | Based on credit usage |
| Self-Hosted | $20-50 | Railway + your time |
| **Savings** | **$150-450** | **75-90% reduction** |

---

## Phase 10: Monitoring & Maintenance

### 10.1 Health Checks

```typescript
// Add to src/api/server.ts
app.get('/health', async (req, res) => {
  try {
    // Check Redis connection
    await redis.ping();
    
    // Check queue stats
    const waiting = await scrapeQueue.getWaitingCount();
    const active = await scrapeQueue.getActiveCount();
    const failed = await scrapeQueue.getFailedCount();
    const completed = await scrapeQueue.getCompletedCount();
    
    // Check database connection
    await pool.query('SELECT 1');
    
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      redis: 'connected',
      database: 'connected',
      queue: {
        waiting,
        active,
        failed,
        completed,
      },
    });
  } catch (error) {
    res.status(500).json({
      status: 'unhealthy',
      error: error.message,
    });
  }
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  const { rows } = await pool.query(`
    SELECT 
      extract_type,
      status,
      COUNT(*) as count,
      AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) as avg_duration_seconds
    FROM scrape_jobs
    WHERE created_at > NOW() - INTERVAL '24 hours'
    GROUP BY extract_type, status
  `);
  
  res.json({
    period: '24 hours',
    metrics: rows,
  });
});
```

### 10.2 Error Notifications

```typescript
// src/notifications/webhook.ts
export async function sendNotification(message: string, severity: 'info' | 'warning' | 'error') {
  if (!process.env.WEBHOOK_URL) return;
  
  await fetch(process.env.WEBHOOK_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: message,
      severity,
      timestamp: new Date().toISOString(),
    }),
  });
}

// Add to worker
worker.on('failed', async (job, err) => {
  console.error(`Job ${job?.id} failed:`, err);
  
  await sendNotification(
    `Scraping job failed: ${job?.data.url}\nError: ${err.message}`,
    'error'
  );
});

// Alert on queue backup
setInterval(async () => {
  const waiting = await scrapeQueue.getWaitingCount();
  
  if (waiting > 100) {
    await sendNotification(
      `Queue backup detected: ${waiting} jobs waiting`,
      'warning'
    );
  }
}, 60000); // Check every minute
```

### 10.3 Analytics Dashboard Queries

```sql
-- Daily scraping summary
SELECT 
  DATE(created_at) as date,
  extract_type,
  COUNT(*) as total_jobs,
  SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) as completed,
  SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) as failed,
  AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) as avg_duration_seconds
FROM scrape_jobs
WHERE created_at > NOW() - INTERVAL '30 days'
GROUP BY DATE(created_at), extract_type
ORDER BY date DESC;

-- Top failing URLs
SELECT 
  url,
  COUNT(*) as failure_count,
  MAX(error) as last_error
FROM scrape_jobs
WHERE status = 'failed'
  AND created_at > NOW() - INTERVAL '7 days'
GROUP BY url
ORDER BY failure_count DESC
LIMIT 10;

-- Schools with missing data
SELECT 
  s.name,
  s.sport,
  MAX(c.scraped_at) as last_coach_scrape,
  MAX(r.scraped_at) as last_roster_scrape
FROM schools s
LEFT JOIN extracted_coaches c ON s.name = c.school_name
LEFT JOIN extracted_rosters r ON s.name = r.school_name
WHERE s.scrape_enabled = true
GROUP BY s.name, s.sport
HAVING 
  MAX(c.scraped_at) IS NULL 
  OR MAX(r.scraped_at) IS NULL
  OR MAX(c.scraped_at) < NOW() - INTERVAL '14 days'
ORDER BY s.name;

-- Data quality metrics
SELECT 
  school_name,
  sport,
  COUNT(*) as total_coaches,
  SUM(CASE WHEN email IS NOT NULL THEN 1 ELSE 0 END) as coaches_with_email,
  SUM(CASE WHEN phone IS NOT NULL THEN 1 ELSE 0 END) as coaches_with_phone,
  ROUND(100.0 * SUM(CASE WHEN email IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*), 2) as email_completeness_pct
FROM extracted_coaches
GROUP BY school_name, sport
ORDER BY email_completeness_pct ASC
LIMIT 20;
```

### 10.4 Automated Alerts

```typescript
// scripts/healthMonitor.ts
import cron from 'node-cron';
import { sendNotification } from './notifications/webhook';
import { pool } from './db/pool';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// Run every hour
cron.schedule('0 * * * *', async () => {
  try {
    // Check for high failure rate
    const { rows } = await pool.query(`
      SELECT 
        COUNT(*) FILTER (WHERE status = 'failed') as failed,
        COUNT(*) as total
      FROM scrape_jobs
      WHERE created_at > NOW() - INTERVAL '1 hour'
    `);
    
    const failureRate = rows[0].failed / rows[0].total;
    
    if (failureRate > 0.2) {
      await sendNotification(
        `High failure rate detected: ${(failureRate * 100).toFixed(1)}% of jobs failing`,
        'error'
      );
    }
    
    // Check worker health
    const activeWorkers = await redis.scard('workers:active');
    if (activeWorkers === 0) {
      await sendNotification(
        'No active workers detected!',
        'error'
      );
    }
    
  } catch (error) {
    await sendNotification(
      `Health check failed: ${error.message}`,
      'error'
    );
  }
});
```

---

## Phase 11: Integration with Athlead

### 11.1 API Client Library

```typescript
// client/firecrawl.ts
export class FirecrawlClient {
  constructor(private baseUrl: string) {}
  
  async scrapeSchool(school: string, sport: string, urls: { coaches: string, roster: string }) {
    const jobs = [];
    
    // Queue coaches
    const coachJob = await this.scrape({
      url: urls.coaches,
      extractType: 'coaches',
      metadata: { schoolName: school, sport },
    });
    jobs.push(coachJob);
    
    // Queue roster
    const rosterJob = await this.scrape({
      url: urls.roster,
      extractType: 'roster',
      metadata: { schoolName: school, sport },
    });
    jobs.push(rosterJob);
    
    return jobs;
  }
  
  async scrape(params: {
    url: string;
    extractType: 'coaches' | 'roster';
    metadata?: any;
  }) {
    const response = await fetch(`${this.baseUrl}/api/scrape`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(params),
    });
    
    if (!response.ok) {
      throw new Error(`Scrape request failed: ${response.statusText}`);
    }
    
    return response.json();
  }
  
  async getStatus(jobId: string) {
    const response = await fetch(`${this.baseUrl}/api/status/${jobId}`);
    
    if (!response.ok) {
      throw new Error(`Status check failed: ${response.statusText}`);
    }
    
    return response.json();
  }
  
  async waitForCompletion(jobId: string, maxWait = 60000) {
    const startTime = Date.now();
    
    while (Date.now() - startTime < maxWait) {
      const status = await this.getStatus(jobId);
      
      if (status.state === 'completed') {
        return status.result;
      }
      
      if (status.state === 'failed') {
        throw new Error(`Job failed: ${status.error}`);
      }
      
      // Wait 2 seconds before checking again
      await new Promise(resolve => setTimeout(resolve, 2000));
    }
    
    throw new Error('Job timeout');
  }
  
  async scrapeAndWait(params: {
    url: string;
    extractType: 'coaches' | 'roster';
    metadata?: any;
  }) {
    const { jobId } = await this.scrape(params);
    return this.waitForCompletion(jobId);
  }
}

// Usage example
const client = new FirecrawlClient('https://your-firecrawl.railway.app');

// Scrape a single school
const jobs = await client.scrapeSchool('Texas Tech', 'Football', {
  coaches: 'https://texastech.com/sports/football/roster/coaches',
  roster: 'https://texastech.com/sports/football/roster',
});

// Wait for results
const [coachesResult, rosterResult] = await Promise.all(
  jobs.map(job => client.waitForCompletion(job.jobId))
);
```

### 11.2 Integration Example

```typescript
// integration/athlead.ts
import { FirecrawlClient } from './client/firecrawl';
import { Pool } from 'pg';

const client = new FirecrawlClient(process.env.FIRECRAWL_URL);
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function syncSchoolData(schoolId: number) {
  // Get school info from Athlead database
  const { rows } = await pool.query(
    'SELECT name, sport, coach_url, roster_url FROM schools WHERE id = $1',
    [schoolId]
  );
  
  const school = rows[0];
  
  console.log(`Syncing ${school.name} - ${school.sport}...`);
  
  // Trigger scraping
  const jobs = await client.scrapeSchool(school.name, school.sport, {
    coaches: school.coach_url,
    roster: school.roster_url,
  });
  
  console.log(`Queued ${jobs.length} jobs for ${school.name}`);
  
  // Wait for completion (or handle async)
  const results = await Promise.all(
    jobs.map(job => client.waitForCompletion(job.jobId))
  );
  
  console.log(`✓ Completed scraping ${school.name}`);
  
  return results;
}

// Sync all schools
async function syncAllSchools() {
  const { rows: schools } = await pool.query(
    'SELECT id FROM schools WHERE scrape_enabled = true ORDER BY name'
  );
  
  for (const school of schools) {
    try {
      await syncSchoolData(school.id);
      // Rate limit between schools
      await new Promise(resolve => setTimeout(resolve, 2000));
    } catch (error) {
      console.error(`Failed to sync school ${school.id}:`, error);
    }
  }
}
```

---

## Phase 12: Advanced Features

### 12.1 Social Media Extraction

```typescript
// src/extractors/socialMedia.ts
export async function extractSocialMedia(page) {
  return await page.evaluate(() => {
    const socialLinks = {
      twitter: null,
      instagram: null,
      facebook: null,
      linkedin: null,
    };
    
    const links = Array.from(document.querySelectorAll('a[href]'));
    
    links.forEach(link => {
      const href = link.href.toLowerCase();
      
      if (href.includes('twitter.com') || href.includes('x.com')) {
        socialLinks.twitter = link.href;
      } else if (href.includes('instagram.com')) {
        socialLinks.instagram = link.href;
      } else if (href.includes('facebook.com')) {
        socialLinks.facebook = link.href;
      } else if (href.includes('linkedin.com')) {
        socialLinks.linkedin = link.href;
      }
    });
    
    return socialLinks;
  });
}
```

### 12.2 Coach Profile Deep Scraping

```typescript
// src/extractors/coachProfile.ts
export async function scrapeCoachProfile(page, profileUrl) {
  await page.goto(profileUrl, { waitUntil: 'networkidle' });
  
  const profile = await page.evaluate(() => {
    return {
      name: document.querySelector('h1, .coach-name')?.textContent?.trim(),
      title: document.querySelector('.title, .position')?.textContent?.trim(),
      bio: document.querySelector('.bio, .biography')?.textContent?.trim(),
      education: Array.from(document.querySelectorAll('.education li, .school'))
        .map(el => el.textContent?.trim()),
      experience: Array.from(document.querySelectorAll('.experience li, .position'))
        .map(el => el.textContent?.trim()),
      email: document.querySelector('a[href^="mailto:"]')?.href?.replace('mailto:', ''),
      phone: document.body.textContent?.match(/\d{3}[-.]?\d{3}[-.]?\d{4}/)?.[0],
    };
  });
  
  const socialMedia = await extractSocialMedia(page);
  
  return {
    ...profile,
    socialMedia,
  };
}

// Update worker to follow coach profile links
async function extractSidearmCoaches(page) {
  const coaches = await page.evaluate(() => {
    const coachCards = [];
    document.querySelectorAll('.coach-card').forEach(card => {
      const link = card.querySelector('a[href*="/coaches/"]');
      coachCards.push({
        name: card.querySelector('.name')?.textContent?.trim(),
        profileUrl: link?.href,
      });
    });
    return coachCards;
  });
  
  // Deep scrape each coach profile
  const detailedCoaches = [];
  for (const coach of coaches) {
    if (coach.profileUrl) {
      try {
        const profile = await scrapeCoachProfile(page, coach.profileUrl);
        detailedCoaches.push(profile);
      } catch (error) {
        console.error(`Failed to scrape profile for ${coach.name}:`, error);
        detailedCoaches.push(coach); // Use basic info
      }
    }
  }
  
  return detailedCoaches;
}
```

### 12.3 Change Detection

```typescript
// src/changeDetection/detector.ts
import { createHash } from 'crypto';

function hashContent(content: any): string {
  return createHash('sha256').update(JSON.stringify(content)).digest('hex');
}

export async function detectChanges(url: string, newData: any) {
  const { rows } = await pool.query(
    'SELECT content_hash FROM scrape_history WHERE url = $1 ORDER BY scraped_at DESC LIMIT 1',
    [url]
  );
  
  const newHash = hashContent(newData);
  
  if (rows.length === 0) {
    // First scrape
    await pool.query(
      'INSERT INTO scrape_history (url, content_hash, scraped_at) VALUES ($1, $2, NOW())',
      [url, newHash]
    );
    return { changed: true, isNew: true };
  }
  
  const oldHash = rows[0].content_hash;
  const changed = oldHash !== newHash;
  
  if (changed) {
    await pool.query(
      'INSERT INTO scrape_history (url, content_hash, scraped_at) VALUES ($1, $2, NOW())',
      [url, newHash]
    );
    
    // Send notification
    await sendNotification(
      `Change detected on ${url}`,
      'info'
    );
  }
  
  return { changed, isNew: false };
}
```

---

## Success Metrics & KPIs

### Key Performance Indicators

**Cost Efficiency:**
- Target: <$50/month total infrastructure
- Comparison: 75-90% savings vs. Firecrawl API

**Performance:**
- Sidearm sites: <10 seconds per page
- Custom sites: <30 seconds per page
- Queue throughput: 100+ jobs/hour per worker
- Uptime: >99%

**Data Quality:**
- Coach contact extraction: >90% accuracy
- Roster data extraction: >85% accuracy
- Error rate: <5%
- Email capture rate: >80% for coaches

**Coverage:**
- Target: 500+ schools
- Sports per school: 10-15
- Total coaches: 5,000+
- Total athletes: 50,000+

### Monitoring Dashboard

Create queries for real-time monitoring:

```sql
-- Real-time queue stats
SELECT 
  'waiting' as status, COUNT(*) as count
FROM bull_jobs
WHERE state = 'waiting'
UNION ALL
SELECT 'active', COUNT(*) FROM bull_jobs WHERE state = 'active'
UNION ALL
SELECT 'completed', COUNT(*) FROM bull_jobs WHERE state = 'completed' AND completed_at > NOW() - INTERVAL '1 hour'
UNION ALL
SELECT 'failed', COUNT(*) FROM bull_jobs WHERE state = 'failed' AND failed_at > NOW() - INTERVAL '1 hour';

-- Coverage metrics
SELECT 
  sport,
  COUNT(DISTINCT school_name) as schools_covered,
  COUNT(*) as total_coaches,
  SUM(CASE WHEN email IS NOT NULL THEN 1 ELSE 0 END) as coaches_with_email
FROM extracted_coaches
GROUP BY sport
ORDER BY schools_covered DESC;
```

---

## Troubleshooting Guide

### Common Issues

**1. Worker not processing jobs**
```bash
# Check Redis connection
redis-cli ping

# Check queue stats
redis-cli LLEN bull:scraping:wait

# Check worker logs
railway logs --service firecrawl-worker
```

**2. Playwright browser crashes**
- Increase worker memory to 2GB
- Check for --no-sandbox flag in launch options
- Verify Playwright dependencies installed

**3. High failure rate**
- Check website structure changes
- Verify selectors are up to date
- Check for rate limiting or IP blocks
- Review error logs for patterns

**4. Slow scraping**
- Increase worker concurrency
- Add more worker replicas
- Implement caching
- Check for network throttling

**5. Database connection issues**
- Verify DATABASE_URL is correct
- Check connection pool limits
- Monitor active connections

### Debug Mode

```typescript
// Add to worker for verbose logging
const worker = new Worker(
  'scraping',
  async (job) => {
    console.log('Job started:', {
      id: job.id,
      url: job.data.url,
      extractType: job.data.extractType,
    });
    
    // ... existing code ...
    
    console.log('Job completed:', {
      id: job.id,
      duration: Date.now() - startTime,
      dataPoints: data.length,
    });
  },
  {
    connection: redis,
    concurrency: 5,
  }
);
```

---

## Next Steps

1. **Week 1-2: Setup & Testing**
   - Fork Firecrawl repo
   - Set up Railway services
   - Test locally with 10-20 schools
   - Deploy to Railway

2. **Week 3: Optimization**
   - Tune extraction selectors
   - Implement caching
   - Add error handling
   - Set up monitoring

3. **Week 4: Scale**
   - Process first 100 schools
   - Monitor performance
   - Adjust worker count
   - Optimize costs

4. **Month 2: Production**
   - Scale to 500+ schools
   - Set up scheduled scraping
   - Implement change detection
   - Build analytics dashboard

5. **Ongoing: Maintenance**
   - Monitor for website changes
   - Update selectors as needed
   - Track data quality metrics
   - Optimize for new CMS patterns

---

## Additional Resources

**Documentation:**
- [Firecrawl GitHub](https://github.com/firecrawl-dev/firecrawl)
- [BullMQ Documentation](https://docs.bullmq.io/)
- [Playwright Documentation](https://playwright.dev/)
- [Railway Documentation](https://docs.railway.app/)

**Tools:**
- [Railway CLI](https://docs.railway.app/develop/cli)
- [Redis CLI](https://redis.io/docs/ui/cli/)
- [Artillery Load Testing](https://www.artillery.io/)

**Support:**
- Railway Discord
- BullMQ GitHub Issues
- Playwright Discord

---

## Conclusion

This plan provides a complete roadmap for deploying a self-hosted Firecrawl system optimized for athletic website scraping. The architecture is scalable, cost-effective, and tailored specifically for your use case.

**Key Benefits:**
- 75-90% cost savings vs. managed services
- Full control and customization
- Scalable architecture
- Production-ready monitoring

**Next Action:** Start with Phase 1 and deploy a proof of concept with 10-20 schools to validate the approach.
