# WEB-CRAWLER

A distributed web crawler pipeline built in Go, split across three independent services that communicate via Kafka.

## Architecture

```
POST /seed
    │
    ▼
crawler-seed (HTTP server)
    │
    ▼
[discovered-urls]  ──►  crawler-worker  ──►  [crawled-urls]  ──►  crawler-parser  ──►  MySQL (url_metadata)
        ▲                     │                                           │
        │               Redis (dedup,                               Redis (dedup:
        │               robots cache,                               parsed_urls)
        │               rate limiter)                                    │
        └───────────────────────────────────────────────────────────────┘
                              (newly discovered links, depth + 1)
```

## Services

| Service | Repo | Description |
|---------|------|-------------|
| **crawler-seed** | [crawler-seed](crawler-seed/) | HTTP server that accepts seed URLs via `POST /seed` and publishes them to `discovered-urls` |
| **crawler-worker** | [crawler-worker](crawler-worker/) | Fetches pages from `discovered-urls`, respects `robots.txt` and rate limits, publishes compressed HTML to `crawled-urls` |
| **crawler-parser** | [crawler-parser](crawler-parser/) | Parses crawled pages, extracts links, stores metadata in MySQL, deduplicates via Redis, and republishes new links to `discovered-urls` |

## Prerequisites

| Dependency | Minimum version | Notes |
|------------|-----------------|-------|
| Go         | 1.25            | |
| Kafka      | 3.x             | Topics `discovered-urls` and `crawled-urls` must exist |
| Redis      | 6.x             | Used by worker and parser for deduplication |
| MySQL      | 8.x             | Used by parser; database `webcrawler` must exist |

### MySQL schema

```sql
CREATE DATABASE IF NOT EXISTS webcrawler;

USE webcrawler;

CREATE TABLE IF NOT EXISTS url_metadata (
  url_hash        CHAR(64)     NOT NULL,
  canonical_url   TEXT         NOT NULL,
  host            VARCHAR(255) NOT NULL,
  status          ENUM('DISCOVERED', 'PARSED', 'FAILED') NOT NULL,
  http_status     SMALLINT     NOT NULL,
  last_crawled_at DATETIME,
  content_hash    CHAR(64)     NOT NULL,
  title           TEXT,
  PRIMARY KEY (url_hash)
);
```

## Getting started

### 1. Clone with submodules

```bash
git clone --recurse-submodules git@github.com:hossainshakhawat/WEB-CRAWLER.git
cd WEB-CRAWLER
```

Or, if you already cloned without submodules:

```bash
git submodule update --init --recursive
```

### 2. Start infrastructure

Ensure Kafka, Redis, and MySQL are running and the required topics exist:

```bash
# Create Kafka topics (adjust broker address as needed)
kafka-topics.sh --create --topic discovered-urls --bootstrap-server localhost:9092
kafka-topics.sh --create --topic crawled-urls    --bootstrap-server localhost:9092
```

### 3. Build and run each service

Each service reads its configuration from a `config.yml` in its working directory and supports environment variable overrides.

```bash
# Terminal 1 — seed service (HTTP server on :8080)
cd crawler-seed && go build -o crawler-seed ./... && ./crawler-seed

# Terminal 2 — worker
cd crawler-worker && go build -o crawler-worker ./... && ./crawler-worker

# Terminal 3 — parser
cd crawler-parser && go build -o crawler-parser ./... && ./crawler-parser
```

### 4. Submit a seed URL

```bash
curl -X POST http://localhost:8080/seed \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com","depth":2}'
```

Response (`202 Accepted`):

```json
{ "published": true, "url": "https://example.com", "depth": 2 }
```

## Configuration summary

Each service uses a `SEED_`, `WORKER_`, or `PARSER_` environment variable prefix. See each service's README for the full configuration reference.

| Service | Key env var prefix | Config file |
|---------|--------------------|-------------|
| crawler-seed   | `SEED_`   | [crawler-seed/config.yml](crawler-seed/config.yml) |
| crawler-worker | `WORKER_` | [crawler-worker/config.yml](crawler-worker/config.yml) |
| crawler-parser | `PARSER_` | [crawler-parser/config.yml](crawler-parser/config.yml) |

## Kafka topics

| Topic | Producer | Consumer | Message type |
|-------|----------|----------|--------------|
| `discovered-urls` | crawler-seed, crawler-parser | crawler-worker | `DiscoveredURL` |
| `crawled-urls`    | crawler-worker               | crawler-parser | `CrawledPage`   |

## Scaling

- **crawler-worker** and **crawler-parser** both support horizontal scaling. Run multiple instances under the same Kafka consumer group (`crawler-workers` / `crawler-parsers`) and Kafka distributes partitions automatically.
- Redis deduplication ensures each URL is only fetched and parsed once across all instances.
