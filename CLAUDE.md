# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Firecrawl is a web scraper API that converts websites into LLM-ready formats (markdown, structured data, HTML). This is a **monorepo** with the following structure:

- **`apps/api`** - Core API server and worker processes (TypeScript/Express)
- **`apps/js-sdk`** - JavaScript/TypeScript SDK
- **`apps/python-sdk`** - Python SDK
- **`apps/rust-sdk`** - Rust SDK
- **`apps/test-suite`** - Shared test infrastructure
- **`apps/playwright-service-ts`** - Playwright-based scraping service
- **`apps/nuq-postgres`** - PostgreSQL queue database setup

## Architecture

### API Structure (`apps/api/src/`)
- **`controllers/`** - API endpoint handlers organized by version (v0, v1, v2)
- **`routes/`** - Express route definitions (v0, v1, v2, admin)
- **`services/`** - Core business logic:
  - `queue-worker.ts` - Main BullMQ job processor
  - `queue-jobs.ts` - Job definitions and scheduling
  - `worker/` - Specialized workers (nuq-worker, nuq-prefetch-worker)
  - `extract-worker.ts` - LLM extraction worker
  - `indexing/` - Search indexing workers
  - `billing/`, `webhook/`, `notification/` - Supporting services
- **`scraper/`** - Web scraping implementation
- **`lib/`** - Shared utilities (concurrency-limit, crawl-redis, extract/, branding/)
- **`__tests__/`** - Test suites, with **`snips/`** containing E2E tests organized by API version

### Key Technologies
- **Express** with WebSocket support (express-ws)
- **BullMQ** for job queuing (backed by Redis and PostgreSQL via NUQ)
- **Redis** for caching, rate limiting, and eviction-based data
- **PostgreSQL** (NUQ) for persistent job queues
- **OpenTelemetry** for observability (otel.ts)
- **Sentry** for error tracking
- **Playwright/Fire-Engine** for JavaScript-rendered pages

## Development Workflow

### Prerequisites
- Node.js, pnpm (v9+), Rust, Redis, PostgreSQL
- Docker (optional, for postgres)
- Set up PostgreSQL using `apps/nuq-postgres/nuq.sql`
- Copy `apps/api/.env.example` to `apps/api/.env` and configure

### Running Locally
**Use the harness for development:**
```bash
cd apps/api
pnpm harness       # Starts API + workers + required services
```

The harness (`src/harness.ts`) orchestrates:
- API server (port 3002 by default)
- Queue workers (BullMQ processors)
- NUQ workers (postgres-backed queuing)
- Extract/index workers
- Postgres container (if using Docker)

**Manual commands (when needed):**
```bash
pnpm dev           # API server only
pnpm workers       # Queue workers only
pnpm nuq-worker    # NUQ worker only
```

### Testing

**E2E Tests (called "snips"):**
Located in `apps/api/src/__tests__/snips/` organized by API version (v1, v2).

**Run tests:**
```bash
pnpm harness jest <pattern>          # Run specific tests with services
pnpm harness jest snips/v2/scrape    # Example: v2 scrape tests
pnpm test                             # Run all tests (slow)
pnpm test:snips                       # Run only snip tests
```

**Important test utilities:**
- **`scrapeTimeout`** from `./lib` - Always use this for test timeouts
- Test gating for different configurations:
  - Fire-engine required: `!process.env.TEST_SUITE_SELF_HOSTED`
  - AI features required: `!process.env.TEST_SUITE_SELF_HOSTED || process.env.OPENAI_API_KEY || process.env.OLLAMA_BASE_URL`

**Run single test file:**
```bash
pnpm harness jest "src/__tests__/snips/v2/scrape.test.ts"
```

### Making Changes to the API

**Standard workflow:**
1. **Write E2E tests first** (`apps/api/src/__tests__/snips/v{1,2}/`)
   - 1+ happy path test(s)
   - 1+ failure path test(s)
   - E2E tests are preferred over unit tests
   - Use `scrapeTimeout` for test timeouts
   - Gate tests appropriately for fire-engine/AI requirements

2. **Implement changes**
   - API endpoints: `src/controllers/v{1,2}/`
   - Routes: `src/routes/v{1,2}/`
   - Business logic: `src/services/` or `src/lib/`
   - Scraping logic: `src/scraper/`

3. **Run tests locally**
   ```bash
   pnpm harness jest <specific-test-pattern>
   ```
   - Run only relevant tests locally (full suite is slow)
   - Let CI run the complete test suite

4. **Push and create PR**
   - CI will run full test suite on multiple configurations
   - Verify all checks pass before merging

### Building
```bash
pnpm build         # Compile TypeScript to dist/
```

### Other Commands
```bash
pnpm format        # Format code with Prettier
pnpm knip          # Check for unused dependencies/exports
```

## Important Conventions

- **E2E over unit tests**: Prefer end-to-end "snip" tests that exercise the full API
- **Use harness for testing**: Don't manually start services; `pnpm harness` handles orchestration
- **API versioning**: Changes to public API should respect existing versions (v1, v2)
- **Redis usage**: Two Redis instances - one for rate limiting (`REDIS_RATE_LIMIT_URL`), one for queuing/caching (`REDIS_URL`)
- **Queue architecture**: Jobs use BullMQ with PostgreSQL persistence (NUQ) for reliability
- **Worker processes**: Multiple worker types handle different job types (scraping, extraction, indexing)