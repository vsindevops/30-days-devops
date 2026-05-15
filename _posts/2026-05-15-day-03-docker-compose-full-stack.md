---
title: "Day 3: Docker Compose for a Full Local Dev Stack — Node.js, PostgreSQL, Redis, and Nginx"
date: 2026-05-15
categories: [Docker]
tags: [docker, docker-compose, postgresql, redis, nginx, nodejs, devops, local-development]
excerpt: "Build a production-realistic local dev environment: Node.js REST API backed by PostgreSQL and Redis, fronted by Nginx — all wired together with health checks, named volumes, and dependency ordering."
header:
  overlay_color: "#0d1117"
  overlay_filter: 0.6
author_profile: false
read_time: true
toc: true
toc_sticky: true
toc_label: "On this page"
---

> **30 Days of DevOps** — a series by [@syssignals](https://x.com/syssignals)
> Every article is a working project. Every command is verified. No fluff.

## The problem with running services one at a time

You have a Node.js app. It needs a database. It needs a cache. It needs a reverse proxy in front of it. In production these run as separate services. In development, most teams run them differently on every laptop: `brew install postgresql`, a Redis Docker container somebody started six months ago, and Nginx… nowhere, because nobody bothered.

The result: bugs that only reproduce in production, onboarding instructions that take three days to follow, and "works on my machine" becoming the team's defining phrase.

Docker Compose was built for exactly this. One `docker-compose.yml` describes the entire stack. One `docker compose up` starts it. Every developer gets an identical environment. Health checks guarantee services start in the right order. Named volumes persist your data across restarts. Bind mounts reload your code without rebuilding the image.

This article builds that environment from scratch. You will leave with a compose file you can adapt to any project in under 10 minutes.

---

## What you'll build

A four-service local development stack:

- **Node.js** — REST API from Day 2, extended to use PostgreSQL and Redis
- **PostgreSQL 16** — primary database with schema migrations run at startup
- **Redis 7** — cache layer with a 60-second TTL on user queries
- **Nginx 1.25** — reverse proxy that terminates HTTP and forwards to the app

Every service has a health check. Nginx only starts when the app is healthy. The app only starts when the database and cache are healthy. No race conditions, no manual retries.

**Estimated time:** 60 minutes

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart LR
    CLIENT(["Browser / curl\nlocalhost:80"]):::host

    subgraph COMPOSE ["docker compose up  —  4 services"]
        NGINX["nginx:1.25-alpine\nReverse Proxy · :80"]:::proxy
        APP["Node.js App\n:3000 (internal)"]:::app
        PG["postgres:16-alpine\n:5432 (internal)"]:::db
        REDIS["redis:7-alpine\n:6379 (internal)"]:::cache
    end

    CLIENT -->|"HTTP :80"| NGINX
    NGINX -->|"proxy_pass :3000"| APP
    APP -->|"SQL queries"| PG
    APP -->|"GET / SET / TTL"| REDIS

    classDef host fill:#1c2128,stroke:#30363d,color:#8b949e
    classDef proxy fill:#1a2744,stroke:#58a6ff,color:#e6edf3
    classDef app fill:#1a2744,stroke:#58a6ff,color:#79b8ff
    classDef db fill:#1f2210,stroke:#3fb950,color:#3fb950
    classDef cache fill:#2a1f10,stroke:#d29922,color:#d29922
```

**Reading this diagram:**

Traffic flows left to right. Start with **"Browser / curl"** on the far left — that's you, or any HTTP client hitting `localhost:80`.

- The request first reaches **Nginx** (blue), the only service with a port binding to your host machine. Nothing else is reachable from outside the Compose network. Nginx's job is to receive the request and forward it on.
- Nginx proxies the request to the **Node.js app** on internal port 3000 (also blue). This port is not exposed to your host — it only exists inside the Docker network. The label `proxy_pass :3000` is the exact Nginx directive that does this forwarding.
- The Node.js app has two backend connections: **PostgreSQL** (green) for persistent data via SQL queries, and **Redis** (yellow) for cache reads and writes with a TTL. Both are internal-only services — they have no host port binding either.

The key point: **only one service faces the outside world**. PostgreSQL and Redis are completely isolated from your laptop. An attacker who finds your laptop's port 80 can only reach Nginx — not your database.

---

## Prerequisites

### Day 2 completed

This article extends the project built in Day 2. You need the `docker-best-practices/` directory on your machine with a working `Dockerfile` and the Node.js application source.

If you skipped Day 2, you can bootstrap the required starting point:

```bash
mkdir docker-best-practices && cd docker-best-practices
npm init -y
mkdir -p src/routes src/middleware

npm install express helmet cors morgan zod dotenv
npm install --save-dev nodemon jest supertest

node -e "
const fs = require('fs');
const pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'));
pkg.main = 'src/index.js';
pkg.engines = { node: '>=20.0.0' };
pkg.scripts = {
  start: 'node src/index.js',
  dev: 'nodemon src/index.js',
  test: 'jest --coverage',
  'test:ci': 'jest --ci --forceExit'
};
fs.writeFileSync('package.json', JSON.stringify(pkg, null, 2));
console.log('package.json updated');
"
```

### Required software

| Tool | Minimum version | Check command |
|---|---|---|
| Docker Engine | 24.0+ | `docker --version` |
| Docker Compose plugin | v2.20+ | `docker compose version` |
| Node.js | 20 LTS | `node --version` |
| npm | 10+ | `npm --version` |
| curl | any | `curl --version` |

Run the full check:

```bash
echo "=== Docker ===" && docker --version && \
echo "=== Compose ===" && docker compose version && \
echo "=== Node ===" && node --version && \
echo "=== npm ===" && npm --version && \
echo "=== curl ===" && curl --version | head -1 && \
echo "" && echo "All prerequisites met."
```

Expected output:

```
=== Docker ===
Docker version 24.0.7, build afdd53b
=== Compose ===
Docker Compose version v2.24.5
=== Node ===
v20.11.0
=== npm ===
10.2.4
=== curl ===
curl 8.4.0 (x86_64-apple-darwin23.0) ...

All prerequisites met.
```

### Operating system

| OS | Status |
|---|---|
| Ubuntu 22.04 / 20.04 | Recommended — all commands tested here |
| macOS 13+ (Sonoma/Ventura) | Supported — use Docker Desktop |
| Windows 11 / 10 (21H2+) | Supported — run all commands in WSL2 |

---

## Project structure for Day 3

Starting from your Day 2 directory, you will add these files:

```
docker-best-practices/
├── src/
│   ├── index.js          ← update: register new routes
│   ├── db.js             ← new: PostgreSQL connection pool
│   ├── cache.js          ← new: Redis client
│   ├── routes/
│   │   ├── health.js     ← update: add /db and /cache checks
│   │   └── users.js      ← update: PostgreSQL + Redis cache-aside
│   └── middleware/
│       └── errorHandler.js
├── db/
│   └── init.sql          ← new: schema + seed data
├── nginx/
│   └── nginx.conf        ← new: reverse proxy config
├── Dockerfile            ← unchanged from Day 2
├── .dockerignore         ← unchanged from Day 2
├── docker-compose.yml    ← replace: full four-service stack
└── .env.example          ← update: add DB and Redis vars
```

---

## Part 1: Extend the application

### Step 1: Install new dependencies

```bash
cd docker-best-practices

# PostgreSQL client
npm install pg

# Redis client (ioredis handles reconnection automatically)
npm install ioredis
```

Verify they landed in `package.json`:

```bash
node -e "
const pkg = require('./package.json');
const deps = Object.keys(pkg.dependencies);
console.log('pg:', deps.includes('pg') ? 'installed' : 'MISSING');
console.log('ioredis:', deps.includes('ioredis') ? 'installed' : 'MISSING');
"
```

Expected output:

```
pg: installed
ioredis: installed
```

### Step 2: PostgreSQL connection pool

Create `src/db.js`:

```javascript
'use strict';

const { Pool } = require('pg');

if (!process.env.DATABASE_URL) {
  throw new Error('DATABASE_URL environment variable is required');
}

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,                    // max connections in pool
  idleTimeoutMillis: 30000,   // close idle connections after 30s
  connectionTimeoutMillis: 3000,
});

pool.on('error', (err) => {
  console.error('Unexpected database pool error:', err.message);
});

/**
 * Run a parameterised query. Always returns from the pool.
 * Usage: const { rows } = await query('SELECT $1::text', ['hello'])
 */
async function query(text, params) {
  const start = Date.now();
  const result = await pool.query(text, params);
  const duration = Date.now() - start;
  if (process.env.NODE_ENV !== 'production') {
    console.debug(`Query: ${text.slice(0, 60)}  [${duration}ms, ${result.rowCount} rows]`);
  }
  return result;
}

module.exports = { pool, query };
```

### Step 3: Redis client

Create `src/cache.js`:

```javascript
'use strict';

const Redis = require('ioredis');

const redis = new Redis(process.env.REDIS_URL || 'redis://localhost:6379', {
  maxRetriesPerRequest: 3,
  retryStrategy(times) {
    // Exponential backoff: 100ms, 200ms, 400ms, …, max 3s
    return Math.min(times * 100, 3000);
  },
  lazyConnect: true,
});

redis.on('connect', () => console.log('Redis: connected'));
redis.on('error',   (err) => console.error('Redis error:', err.message));

module.exports = { redis };
```

### Step 4: Update the health route

Replace `src/routes/health.js` to add `/health/db` and `/health/cache` endpoints:

```javascript
'use strict';

const { Router } = require('express');
const { pool }   = require('../db');
const { redis }  = require('../cache');

const router = Router();

// Basic liveness — does not check dependencies
router.get('/', (req, res) => {
  res.json({
    status:      'healthy',
    timestamp:   new Date().toISOString(),
    uptime:      Math.floor(process.uptime()),
    environment: process.env.NODE_ENV || 'development',
  });
});

router.get('/ready', (req, res) => res.json({ status: 'ready' }));

// Deep health: PostgreSQL
router.get('/db', async (req, res) => {
  try {
    const { rows } = await pool.query('SELECT 1 AS ok');
    res.json({ status: 'healthy', result: rows[0] });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', error: err.message });
  }
});

// Deep health: Redis
router.get('/cache', async (req, res) => {
  try {
    const pong = await redis.ping();
    res.json({ status: 'healthy', response: pong });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', error: err.message });
  }
});

module.exports = { healthRouter: router };
```

### Step 5: Update the users route to use PostgreSQL + Redis

Replace `src/routes/users.js`:

```javascript
'use strict';

const { Router } = require('express');
const { z }      = require('zod');
const { query }  = require('../db');
const { redis }  = require('../cache');

const router = Router();

const CACHE_TTL = 60; // seconds
const CACHE_KEY = 'users:all';

const UserSchema = z.object({
  name:  z.string().min(1).max(100),
  email: z.string().email(),
  role:  z.enum(['admin', 'user', 'viewer']).default('user'),
});

// GET /api/users — cache-aside: check Redis, fall back to PostgreSQL
router.get('/', async (req, res) => {
  // 1. Try cache
  const cached = await redis.get(CACHE_KEY).catch(() => null);
  if (cached) {
    return res.json({ users: JSON.parse(cached), source: 'cache' });
  }

  // 2. Cache miss — query database
  const { rows } = await query(
    'SELECT id, name, email, role, created_at FROM users ORDER BY id'
  );

  // 3. Populate cache for next request
  await redis.setex(CACHE_KEY, CACHE_TTL, JSON.stringify(rows)).catch(() => null);

  res.json({ users: rows, source: 'database' });
});

// GET /api/users/:id
router.get('/:id', async (req, res) => {
  const { rows } = await query(
    'SELECT id, name, email, role, created_at FROM users WHERE id = $1',
    [req.params.id]
  );
  if (rows.length === 0) {
    return res.status(404).json({ error: 'User not found' });
  }
  res.json(rows[0]);
});

// POST /api/users
router.post('/', async (req, res) => {
  const result = UserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({ error: 'Validation failed', details: result.error.flatten() });
  }
  const { name, email, role } = result.data;
  const { rows } = await query(
    'INSERT INTO users (name, email, role) VALUES ($1, $2, $3) RETURNING *',
    [name, email, role]
  );

  // Bust the list cache so the next GET reflects the new user
  await redis.del(CACHE_KEY).catch(() => null);

  res.status(201).json(rows[0]);
});

module.exports = { usersRouter: router };
```

### Step 6: Update index.js to connect on startup

Replace `src/index.js`:

```javascript
'use strict';

const express = require('express');
const helmet  = require('helmet');
const cors    = require('cors');
const morgan  = require('morgan');

const { healthRouter } = require('./routes/health');
const { usersRouter }  = require('./routes/users');
const { errorHandler } = require('./middleware/errorHandler');
const { pool }         = require('./db');
const { redis }        = require('./cache');

const app  = express();
const PORT = process.env.PORT || 3000;

app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost'],
}));
app.use(morgan(process.env.NODE_ENV === 'production' ? 'combined' : 'dev'));
app.use(express.json({ limit: '10kb' }));

app.use('/health',     healthRouter);
app.use('/api/users',  usersRouter);

app.use((req, res) => res.status(404).json({ error: 'Route not found' }));
app.use(errorHandler);

async function start() {
  // Verify database connection before accepting traffic
  await pool.query('SELECT 1');
  console.log('PostgreSQL: connected');

  // Verify Redis connection
  await redis.connect();
  console.log('Redis: ready');

  const server = app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT} [${process.env.NODE_ENV || 'development'}]`);
  });

  const shutdown = (signal) => {
    console.log(`${signal} received — shutting down`);
    server.close(async () => {
      await pool.end();
      redis.disconnect();
      process.exit(0);
    });
  };

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT',  () => shutdown('SIGINT'));
}

start().catch((err) => {
  console.error('Failed to start:', err.message);
  process.exit(1);
});

module.exports = { app };
```

---

## Part 2: Database schema and seed data

Create the `db/` directory and the init script that PostgreSQL runs automatically on first start:

```bash
mkdir -p db
```

Create `db/init.sql`:

```sql
-- Users table
CREATE TABLE IF NOT EXISTS users (
  id         SERIAL       PRIMARY KEY,
  name       VARCHAR(100) NOT NULL,
  email      VARCHAR(255) NOT NULL UNIQUE,
  role       VARCHAR(20)  NOT NULL DEFAULT 'user'
               CHECK (role IN ('admin', 'user', 'viewer')),
  created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- Seed data — ON CONFLICT DO NOTHING makes this safe to re-run
INSERT INTO users (name, email, role) VALUES
  ('Alice', 'alice@example.com', 'admin'),
  ('Bob',   'bob@example.com',   'user'),
  ('Carol', 'carol@example.com', 'viewer')
ON CONFLICT (email) DO NOTHING;

-- Index on email for lookups
CREATE INDEX IF NOT EXISTS users_email_idx ON users (email);
```

> PostgreSQL runs every file in `/docker-entrypoint-initdb.d/` in alphabetical order — but only on the **first** start when the data directory is empty. Subsequent restarts skip the init scripts. This means seed data persists across restarts once written.

---

## Part 3: Nginx reverse proxy config

Create the `nginx/` directory:

```bash
mkdir -p nginx
```

Create `nginx/nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    # Upstream block — Docker DNS resolves "app" to the app container's IP
    upstream app_server {
        server app:3000;
        keepalive 32;
    }

    # Rate limiting — 100 requests/second per IP, burst of 20
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;

    server {
        listen 80;
        server_name localhost;

        # Security headers
        add_header X-Frame-Options        DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection       "1; mode=block";

        # Hide Nginx version from responses
        server_tokens off;

        # Proxy all requests to the Node app
        location / {
            limit_req zone=api burst=20 nodelay;

            proxy_pass         http://app_server;
            proxy_http_version 1.1;

            # Required headers for the app to know the real client
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Keep connections alive to the upstream
            proxy_set_header Connection "";

            proxy_read_timeout    90s;
            proxy_connect_timeout 10s;
            proxy_send_timeout    90s;
        }

        # Health check endpoint — skip access logs so they don't fill up
        location = /health {
            proxy_pass http://app_server/health;
            access_log off;
        }
    }
}
```

---

## Part 4: Environment variables

Update `.env.example`:

```bash
cat > .env.example << 'EOF'
# Application
NODE_ENV=development
PORT=3000
ALLOWED_ORIGINS=http://localhost

# PostgreSQL — matches the postgres service credentials in docker-compose.yml
DATABASE_URL=postgresql://appuser:apppassword@postgres:5432/appdb

# Redis
REDIS_URL=redis://redis:6379
EOF
```

Create your local `.env` from the example:

```bash
cp .env.example .env
```

> **Never commit `.env` to git.** It is already in `.gitignore` from Day 2. The `.env.example` is safe to commit — it documents required variables without containing real values.

---

## Part 5: The complete docker-compose.yml

This is the heart of the article. Replace the existing `docker-compose.yml` entirely:

```yaml
version: '3.9'

# ─── Services ─────────────────────────────────────────────────────────────────

services:

  # ── Nginx ──────────────────────────────────────────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    container_name: dev-nginx
    restart: unless-stopped
    ports:
      - "80:80"             # Only Nginx is exposed to the host
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      app:
        condition: service_healthy  # Wait until app passes its health check
    networks:
      - app-network

  # ── Node.js Application ────────────────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: deps           # dev target: includes all dependencies
    container_name: dev-app
    restart: unless-stopped
    expose:
      - "3000"               # Internal only — Nginx proxies to this
    volumes:
      - ./src:/app/src:ro    # Bind mount: file changes without rebuild
      - node_modules:/app/node_modules
    env_file:
      - .env
    environment:
      NODE_ENV: development
    command: ["node_modules/.bin/nodemon", "--watch", "src", "src/index.js"]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test:
        - CMD
        - node
        - -e
        - >
          require('http')
            .get('http://localhost:3000/health', r => process.exit(r.statusCode===200?0:1))
            .on('error', () => process.exit(1))
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - app-network

  # ── PostgreSQL ─────────────────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    container_name: dev-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB:       appdb
      POSTGRES_USER:     appuser
      POSTGRES_PASSWORD: apppassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/01_init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - app-network

  # ── Redis ──────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: dev-redis
    restart: unless-stopped
    command:
      - redis-server
      - --save          # Snapshot: if 1 key changed in 60s, write to disk
      - "60"
      - "1"
      - --loglevel
      - warning
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 10s
    networks:
      - app-network

# ─── Volumes ──────────────────────────────────────────────────────────────────

volumes:
  postgres_data:    # PostgreSQL data files — persist across container restarts
  redis_data:       # Redis RDB snapshots — cache survives restarts
  node_modules:     # Prevents host node_modules from shadowing container ones

# ─── Networks ─────────────────────────────────────────────────────────────────

networks:
  app-network:
    driver: bridge  # Default bridge with automatic DNS — containers reach each
                    # other by service name (postgres, redis, app, nginx)
```

---

## Part 6: Service startup ordering and health checks

This is the most important concept in the entire article. Without it, your app starts before the database is ready, crashes on the first query, and the container exits.

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart TD
    START(["docker compose up"]):::entry

    subgraph WAVE1 ["Wave 1 — start immediately"]
        PG["postgres:16-alpine\nwait for pg_isready"]:::db
        RD["redis:7-alpine\nwait for redis-cli ping"]:::cache
    end

    subgraph WAVE2 ["Wave 2 — start when Wave 1 healthy"]
        APP["Node.js app\nwait for HTTP 200 on /health"]:::app
    end

    subgraph WAVE3 ["Wave 3 — start when Wave 2 healthy"]
        NGX["nginx:1.25-alpine\nready to accept traffic"]:::proxy
    end

    START --> WAVE1
    PG  -->|"service_healthy"| WAVE2
    RD  -->|"service_healthy"| WAVE2
    APP -->|"service_healthy"| WAVE3

    classDef entry fill:#1a2744,stroke:#58a6ff,color:#e6edf3
    classDef db    fill:#1f2210,stroke:#3fb950,color:#3fb950
    classDef cache fill:#2a1f10,stroke:#d29922,color:#d29922
    classDef app   fill:#1a2744,stroke:#58a6ff,color:#79b8ff
    classDef proxy fill:#1c2128,stroke:#30363d,color:#8b949e
```

**Reading this diagram:**

Read top to bottom. `docker compose up` triggers three waves of startup — each wave waits for the previous one to complete before starting.

- **Wave 1 (green + yellow):** PostgreSQL and Redis start immediately in parallel. Neither has dependencies. Each runs its own health check on a loop: PostgreSQL uses `pg_isready`, Redis uses `redis-cli ping`. Until these pass, no other service starts.
- **Wave 2 (blue):** The Node.js app starts *only after both* PostgreSQL and Redis report `service_healthy`. The `depends_on: condition: service_healthy` entries in the compose file enforce this. The app then runs its own health check — an HTTP request to `/health` — before it is considered ready.
- **Wave 3 (grey):** Nginx starts *only after* the app is healthy. By this point the full chain is verified: database is up, cache is up, app is connected to both and responding on `/health`. Nginx can safely proxy traffic without hitting a "connection refused" error.

The `service_healthy` condition replaces the old pattern of `sleep 5 && start-app` scripts, which are inherently fragile. This approach is deterministic: the wait is based on *actual readiness*, not a guess.

The `depends_on` with `condition: service_healthy` means Docker Compose will not start the dependent service until the dependency's health check returns a passing state. This replaces the old pattern of `sleep 5 && start-app` shell scripts.

Each health check has four parameters:

| Parameter | PostgreSQL | Redis | App |
|---|---|---|---|
| `interval` | 10s | 10s | 15s |
| `timeout` | 5s | 3s | 5s |
| `retries` | 5 | 3 | 5 |
| `start_period` | 30s | 10s | 30s |

`start_period` is critical: failures during this window don't count towards `retries`. It gives the service time to initialise without the container being marked unhealthy before it has even finished starting.

---

## Part 7: Named volumes and data persistence

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart LR
    subgraph HOST ["Docker Host — persists across restarts"]
        PGV[("postgres_data\nnamed volume")]:::vol
        RDV[("redis_data\nnamed volume")]:::vol
        NMV[("node_modules\nnamed volume")]:::vol
        SRC["./src\nbind mount"]:::bind
        NGINX_CONF["./nginx/nginx.conf\nbind mount  :ro"]:::bind
    end

    subgraph CONTAINERS ["Containers"]
        PGC["/var/lib/postgresql/data"]:::svc
        RDC["/data"]:::svc
        NMC["/app/node_modules"]:::svc
        SRCC["/app/src  (read-only)"]:::svc
        NGXC["/etc/nginx/nginx.conf  (ro)"]:::svc
    end

    PGV      <-->|"mounted at"| PGC
    RDV      <-->|"mounted at"| RDC
    NMV      <-->|"mounted at"| NMC
    SRC      <-->|"live sync"| SRCC
    NGINX_CONF <-->|"live sync"| NGXC

    classDef vol  fill:#1f2210,stroke:#3fb950,color:#3fb950
    classDef bind fill:#1a2744,stroke:#58a6ff,color:#79b8ff
    classDef svc  fill:#1c2128,stroke:#30363d,color:#8b949e
```

**Reading this diagram:**

The left side is the Docker host (your machine). The right side is the running containers. Every double-headed arrow is a mount — a path inside the container that is backed by storage outside it.

There are two types, shown by colour:

- **Green cylinders (named volumes):** `postgres_data`, `redis_data`, and `node_modules` are managed entirely by Docker. You don't see them as folders on your filesystem — Docker stores them in its own internal directory. They survive `docker compose down` but are wiped by `docker compose down -v`. Use these for anything that must *persist* (database files, cache snapshots) and for `node_modules` where host/container compatibility matters.
- **Blue rectangles (bind mounts):** `./src` and `./nginx/nginx.conf` are actual folders and files on your laptop. Docker maps them directly into the container path. A file save on your host is instantly visible inside the container — this is what makes hot reload work. The `:ro` flag on `nginx.conf` means the container can read the config but cannot write back to it.

The `node_modules` named volume deserves special attention. If you bind-mounted your entire project root, your host's `node_modules` (compiled for macOS or Windows) would overwrite the container's `node_modules` (compiled for Alpine Linux). Native addons like `bcrypt` would silently break. The named volume at `/app/node_modules` takes precedence over any bind mount at that path, keeping the two copies completely separate.

**Named volumes** (`postgres_data`, `redis_data`, `node_modules`) are managed by Docker. They survive `docker compose down` but are removed by `docker compose down -v`. Use named volumes for anything you want to persist.

**Bind mounts** (`./src`, `./nginx/nginx.conf`) link a path on your host to a path inside the container. Changes on the host are immediately visible inside the container — enabling hot reload without rebuilding the image.

The `node_modules` volume deserves explanation: without it, `COPY . .` in the Dockerfile and `./src:/app/src` bind mount would conflict with `npm install` output. The named volume shadows the bind-mount path for `node_modules` specifically, keeping host and container copies separate.

---

## Part 8: The cache-aside pattern in action

The `/api/users` route uses cache-aside (also called lazy loading). The first request hits PostgreSQL and populates Redis. Every subsequent request within the 60-second TTL is served from Redis.

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart TD
    REQ(["GET /api/users"]):::entry
    CHECK{"Redis: GET\nusers:all"}:::decision

    HIT["Return cached JSON\nsource: 'cache'\n~1ms"]:::fast
    MISS["Query PostgreSQL\nSELECT * FROM users\n~5–20ms"]:::slow
    STORE["Redis: SETEX\nusers:all  TTL 60s"]:::store
    RESP(["HTTP 200 to client"]):::entry

    REQ --> CHECK
    CHECK -->|"cache hit"| HIT
    CHECK -->|"cache miss"| MISS
    MISS --> STORE
    STORE --> RESP
    HIT --> RESP

    classDef entry    fill:#1a2744,stroke:#58a6ff,color:#e6edf3
    classDef decision fill:#2a1f10,stroke:#d29922,color:#d29922
    classDef fast     fill:#0d2818,stroke:#3fb950,color:#3fb950
    classDef slow     fill:#1a2744,stroke:#58a6ff,color:#79b8ff
    classDef store    fill:#2a1f10,stroke:#d29922,color:#e6edf3
```

**Reading this diagram:**

Every `GET /api/users` request takes one of two paths depending on whether Redis has a cached copy.

- **The diamond (yellow)** is the decision point — the app runs `redis.get('users:all')`. Two outcomes:
- **Cache hit (green path, left):** Redis has the data. It's returned immediately as JSON — no database involved. Response time is ~1ms. This is the happy path for the second, third, fourth… request within the 60-second window.
- **Cache miss (blue path, right):** Redis has nothing (first request, or TTL expired). The app falls through to PostgreSQL with a `SELECT * FROM users` query. This takes ~5–20ms depending on load. After the query returns, the result is immediately written into Redis with a 60-second TTL (`SETEX`) so the next request gets the fast path.

Both paths converge at the same **HTTP 200 response** to the client. The only difference is *where* the data came from — and the response includes a `source` field (`"cache"` or `"database"`) so you can observe this behaviour directly with `curl`.

The cache is **invalidated on writes**: when `POST /api/users` creates a new user, the route calls `redis.del('users:all')`. The next list request gets a cache miss, re-queries PostgreSQL with the new user included, and repopulates the cache. The data is never stale for more than one write cycle.

When a `POST /api/users` creates a new user, the route immediately calls `redis.del('users:all')`, so the next list request re-queries PostgreSQL and repopulates the cache with fresh data.

---

## Part 9: Run and verify the complete stack

### Start the stack

```bash
docker compose up --build
```

Expected output (abbreviated — services start in dependency order):

```
[+] Building 22.1s (11/11) FINISHED
[+] Running 5/5
 ✔ Network docker-best-practices_app-network  Created
 ✔ Container dev-postgres                     Started
 ✔ Container dev-redis                        Started
 ✔ Container dev-app                          Started
 ✔ Container dev-nginx                        Started
dev-postgres  | PostgreSQL init process complete; ready for start up.
dev-redis     | 1:M * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
dev-app       | PostgreSQL: connected
dev-app       | Redis: ready
dev-app       | Server running on port 3000 [development]
```

### Check all services are healthy

Open a second terminal:

```bash
docker compose ps
```

Expected output:

```
NAME           IMAGE                  STATUS                   PORTS
dev-nginx      nginx:1.25-alpine      Up 12 seconds            0.0.0.0:80->80/tcp
dev-app        docker-best-p-app      Up 23 seconds (healthy)  3000/tcp
dev-postgres   postgres:16-alpine     Up 35 seconds (healthy)  5432/tcp
dev-redis      redis:7-alpine         Up 35 seconds (healthy)  6379/tcp
```

All four services must show `healthy` before you continue.

### Test every endpoint

```bash
# Liveness check through Nginx
curl -s http://localhost/health | python3 -m json.tool
```

Expected:

```json
{
    "status": "healthy",
    "timestamp": "2026-05-15T10:23:45.123Z",
    "uptime": 12,
    "environment": "development"
}
```

```bash
# Deep database check
curl -s http://localhost/health/db | python3 -m json.tool
```

Expected:

```json
{
    "status": "healthy",
    "result": {
        "ok": 1
    }
}
```

```bash
# Deep Redis check
curl -s http://localhost/health/cache | python3 -m json.tool
```

Expected:

```json
{
    "status": "healthy",
    "response": "PONG"
}
```

```bash
# List users — first call hits PostgreSQL
curl -s http://localhost/api/users | python3 -m json.tool
```

Expected:

```json
{
    "users": [
        {
            "id": 1,
            "name": "Alice",
            "email": "alice@example.com",
            "role": "admin",
            "created_at": "2026-05-15T10:23:20.000Z"
        },
        {
            "id": 2,
            "name": "Bob",
            "email": "bob@example.com",
            "role": "user",
            "created_at": "2026-05-15T10:23:20.000Z"
        }
    ],
    "source": "database"
}
```

```bash
# Second call — served from Redis cache
curl -s http://localhost/api/users | python3 -m json.tool
```

Notice `"source": "cache"` in the response — Redis answered instead of PostgreSQL.

```bash
# Create a new user
curl -s -X POST http://localhost/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Dave","email":"dave@example.com","role":"viewer"}' \
  | python3 -m json.tool
```

Expected:

```json
{
    "id": 4,
    "name": "Dave",
    "email": "dave@example.com",
    "role": "viewer",
    "created_at": "2026-05-15T10:25:11.000Z"
}
```

```bash
# Cache was busted — this hits PostgreSQL again and returns all 4 users
curl -s http://localhost/api/users | python3 -m json.tool
```

### Test hot reload

Edit a source file without rebuilding:

```bash
# In a second terminal — change the health response
sed -i "s/'healthy'/'all-systems-go'/" src/routes/health.js

# Nodemon restarts automatically — watch the compose logs
# Then verify
curl -s http://localhost/health | grep status
# "status": "all-systems-go"

# Revert
sed -i "s/'all-systems-go'/'healthy'/" src/routes/health.js
```

### Verify data persistence across restarts

```bash
# Restart the app container only (not the database)
docker compose restart app

# Wait for it to be healthy again
docker compose ps

# Users are still there — PostgreSQL data persisted
curl -s http://localhost/api/users | python3 -m json.tool
```

### Connect directly to PostgreSQL (optional)

```bash
docker exec -it dev-postgres psql -U appuser -d appdb
```

Once inside:

```sql
-- List all users
SELECT id, name, email, role FROM users;

-- Check the index was created
\d users

-- Exit
\q
```

### Connect directly to Redis (optional)

```bash
docker exec -it dev-redis redis-cli
```

Once inside:

```
127.0.0.1:6379> KEYS *
1) "users:all"

127.0.0.1:6379> TTL users:all
(integer) 47

127.0.0.1:6379> GET users:all
"[{\"id\":1,\"name\":\"Alice\"..."

127.0.0.1:6379> EXIT
```

### Tear down

```bash
# Stop containers — volumes preserved
docker compose down

# Stop containers AND remove all volumes (fresh start)
docker compose down -v
```

---

## Common errors and fixes

### Error 1: App crashes immediately — `DATABASE_URL environment variable is required`

```
dev-app | Error: DATABASE_URL environment variable is required
dev-app | exited with code 1
```

**Cause:** The `.env` file is missing or not in the same directory as `docker-compose.yml`.

**Fix:**

```bash
# Confirm the file exists
ls -la .env

# If it doesn't, create it from the example
cp .env.example .env

# Confirm DATABASE_URL is set (should not be blank)
grep DATABASE_URL .env
```

---

### Error 2: `depends_on` service is healthy but app still can't connect

```
dev-app | Error: connect ECONNREFUSED 172.18.0.3:5432
```

**Cause:** The health check passes before PostgreSQL is fully ready to accept TCP connections (rare but possible on slow machines).

**Fix:** Increase `start_period` for the PostgreSQL health check:

```yaml
healthcheck:
  start_period: 60s   # was 30s
```

---

### Error 3: `port is already allocated` on port 80

```
Error response from daemon: Ports are not available: exposing port TCP 0.0.0.0:80
```

**Cause:** Another process (Apache, another Nginx, or a previous compose run) is using port 80.

**Fix:**

```bash
# Find what's on port 80
sudo lsof -i :80

# If it's a previous compose run
docker compose down

# Or change the Nginx port in docker-compose.yml to something free
ports:
  - "8080:80"
# Then access the app at http://localhost:8080
```

---

### Error 4: PostgreSQL init.sql not running

```
# Users table doesn't exist
dev-app | error: relation "users" does not exist
```

**Cause:** PostgreSQL only runs init scripts on first start (when the data directory is empty). If you previously started the postgres container without the init script, the volume has data and the script is skipped.

**Fix:** Remove the postgres volume to force a fresh initialisation:

```bash
docker compose down -v          # -v removes named volumes
docker compose up --build       # fresh start — init.sql will run
```

---

### Error 5: Redis connection refused

```
dev-app | Redis error: connect ECONNREFUSED 127.0.0.1:6379
```

**Cause:** The app is using `localhost` for Redis instead of the service name `redis`.

**Fix:** Ensure `REDIS_URL` in `.env` uses the Docker service name:

```bash
# Wrong
REDIS_URL=redis://localhost:6379

# Correct — "redis" resolves to the redis container via Docker DNS
REDIS_URL=redis://redis:6379
```

---

### Error 6: Hot reload not working on Linux

```
[nodemon] watching: src/**/*
# (no restart when files change)
```

**Cause:** Linux kernel doesn't propagate inotify events to containers by default in some configurations.

**Fix:** Add `--legacy-watch` to the nodemon command in `docker-compose.yml`:

```yaml
command: ["node_modules/.bin/nodemon", "--legacy-watch", "--watch", "src", "src/index.js"]
```

---

## How it works under the hood

### Docker DNS and service discovery

Every service in a Compose network can reach every other service by its service name. When the app calls `postgresql://appuser:apppassword@postgres:5432/appdb`, Docker's embedded DNS server resolves `postgres` to the IP address of the `dev-postgres` container. No `/etc/hosts` entries required, no hardcoded IPs, no service registry.

This works because all four services are on the same named network (`app-network`). If you remove the `networks` key, Compose creates a default network and the same DNS resolution applies — but explicit networks make it clear which services can talk to each other.

### Why only Nginx is exposed on a host port

Only `nginx` has a `ports:` entry (`80:80`). The other three services use `expose:` (app) or nothing at all (postgres, redis). `expose:` documents that a port is available inside the network but does not bind it to the host.

This means from your laptop, you can only reach the stack via `http://localhost`. You cannot directly connect a local psql client to `localhost:5432` — there is no host port binding. This is intentional: it prevents accidentally connecting tools to the dev database when you think you're connected to a remote one. If you need direct database access, use `docker exec -it dev-postgres psql`.

### Why node_modules is a named volume

When you mount `./src:/app/src`, Docker uses the host filesystem for that path. If you also mounted `./:/app`, your host's `node_modules` (compiled for macOS or Windows) would override the container's `node_modules` (compiled for Linux). Packages with native addons would break silently.

The named volume `node_modules:/app/node_modules` takes precedence over any bind mount at the same path. The container installs its own platform-correct node_modules into the volume, and the bind mount of `./src` never touches it.

---

## Day 3 recap

You now have:

- A four-service Docker Compose stack that starts with one command
- Health checks on every service with proper start periods and retry logic
- Dependency ordering that eliminates race conditions between services
- PostgreSQL with schema migrations that run automatically on first start
- Redis cache-aside pattern that reduces database load on repeated queries
- Nginx reverse proxy with rate limiting and security headers
- Named volumes that persist data across restarts
- Bind mounts that deliver hot reload without image rebuilds
- Direct database and cache access via `docker exec` for debugging

---

## Day 4 preview

**Day 4: GitHub Actions CI/CD — build, test, scan, and push on every PR**

A GitHub Actions pipeline that builds the Docker image, runs the test stage as a CI gate, scans with Docker Scout for CVEs, and pushes to GitHub Container Registry on merge to main. The pipeline that runs in 90 seconds and blocks bad code from reaching production.

---

*This is Day 3 of the [30 Days of DevOps](https://x.com/syssignals) series.*
*Follow [@syssignals](https://x.com/syssignals) on X — Day 4 drops tomorrow.*
*Found a command that doesn't work? Reply on X with your OS and Docker version.*
