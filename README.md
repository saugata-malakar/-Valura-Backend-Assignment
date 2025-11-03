# -Valura-Backend-Assignment
Table of contents

Prerequisites

Build & Run Locally (Docker Compose)

Run Tests & Load Tests

Design Document (1–2 pages)

API Endpoints — curl examples + Postman collection

Short Load Test Report & Production Scaling Plan (1 page)

Appendix: Useful commands & tips

Prerequisites

Install these tools locally:

Docker & Docker Compose (v2+)

Go 1.20+ (if running compiled binary locally)

Node.js / npm (optional; for web UI or scripts)

k6 (for load tests) — https://k6.io

vegeta (optional alternative) — https://github.com/tsenart/vegeta

curl / Postman (for API testing)

Build & Run Locally (Docker Compose)

Place the file docker-compose.yml at the repo root. Example:

version: "3.8"
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: exchange
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  nats:
    image: nats:2.9
    ports:
      - "4222:4222"
      - "8222:8222"

  matcher:
    build:
      context: .
      dockerfile: Dockerfile
    depends_on:
      - postgres
      - redis
      - nats
    environment:
      - DATABASE_URL=postgres://postgres:password@postgres:5432/exchange?sslmode=disable
      - REDIS_URL=redis://redis:6379
      - NATS_URL=nats://nats:4222
      - MATCHER_SHARD_COUNT=4
    ports:
      - "8080:8080"
    command: ["./matcher", "--config=/app/config.yml"]

volumes:
  pgdata:


Sample Dockerfile (Go):

FROM golang:1.20-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o matcher ./cmd/matcher

FROM alpine:3.18
RUN apk add --no-cache ca-certificates
WORKDIR /app
COPY --from=builder /src/matcher /app/matcher
COPY config.yml /app/config.yml
CMD ["/app/matcher", "--config=/app/config.yml"]


Quickstart:

# Build images and start services
docker compose up --build

# Wait until Postgres/NATS are up, then initialize schema (example)
docker compose exec matcher /app/matcher migrate

# Open API at:
http://localhost:8080


If you prefer to run the matcher locally (without Docker):

# Ensure Postgres/NATS/Redis are running (via Docker)
go build -o matcher ./cmd/matcher
./matcher --config=config.yml

Run Tests & Load Tests
Unit tests (Go)

Run unit tests locally:

# from repo root
go test ./... -v


Run unit tests inside Docker (integration):

docker compose run --rm matcher go test ./... -v

Integration tests

Integration tests assume services (Postgres, NATS, Redis) are running via Docker Compose.

docker compose up -d postgres redis nats
# then run integration tests that hit the real stack:
docker compose run --rm matcher go test ./integration -v

Load tests

Two example load test tools are included: k6 (scripted scenario) and vegeta (simple attack).

k6 (recommended for realistic scenarios)

Create file loadtests/k6_submit_orders.js:

import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend } from 'k6/metrics';

export let lat = new Trend('latency');

export let options = {
  stages: [
    { duration: '30s', target: 200 },   // ramp up to 200 VUs
    { duration: '2m', target: 200 },
    { duration: '30s', target: 0 },
  ],
};

const API = __ENV.API || 'http://localhost:8080';

export default function () {
  const payload = JSON.stringify({
    clientOrderId: Math.random().toString(36).slice(2),
    symbol: 'BTCUSDT',
    side: Math.random() > 0.5 ? 'BUY' : 'SELL',
    type: 'LIMIT',
    price: (70000 + Math.random()*1000).toFixed(2),
    quantity: (0.001 + Math.random()*0.01).toFixed(6)
  });
  const params = { headers: { 'Content-Type': 'application/json' } };
  let res = http.post(`${API}/api/v1/orders`, payload, params);
  lat.add(res.timings.duration);
  check(res, { 'status 200': (r) => r.status === 200 });
  sleep(0.01);
}


Run k6:

k6 run loadtests/k6_submit_orders.js --vus 200 --duration 2m
# or with environment API var:
API=http://localhost:8080 k6 run loadtests/k6_submit_orders.js

Vegeta example

Generate a target file loadtests/vegeta_targets.txt:

POST http://localhost:8080/api/v1/orders
Content-Type: application/json

{"clientOrderId":"<replace>", "symbol":"BTCUSDT","side":"BUY","type":"LIMIT","price":"70000","quantity":"0.005"}


Then run:

vegeta attack -duration=1m -rate=200 -workers=50 -targets=loadtests/vegeta_targets.txt | vegeta report


Tune rates/VUs/shards to reach target throughput (e.g., 2k orders/sec). See the [Scaling section below] for production steps.

Design Document (1–2 pages)
1. System Overview

This system is a horizontally-scalable matching engine that supports price-time priority, deterministic replay, and real-time streaming. The major components:

API Ingestion layer (HTTP & WebSocket): Accepts orders/cancels/status queries; performs light validation and forwards to partitioned matcher shards.

Matcher shards: Each shard owns a set of symbols (sharding: hash(symbol) % SHARD_COUNT) and runs a single-writer actor to process orders serially.

Persistence:

WAL (append-only ledger) stored in Postgres: every order event and trade is appended (immutable).

Snapshots stored periodically (Postgres blob / S3): capture in-memory book state for fast recovery.

Streaming: NATS JetStream broadcasts book.delta.<symbol> and trades.<symbol> for downstream consumers and UI clients.

Cache: Redis for ephemeral lookups (order ID → shard mapping, order status) and light-weight rate-limiting.

2. Concurrency Model

Shard actor model: Orders are routed to shard-specific goroutines. Each shard has a serialized event queue; the shard processes one event at a time (single-writer), which preserves price-time ordering and avoids fine-grained locking in the hot path.

Frontend workers: API workers validate and enqueue requests to the appropriate shard. If the shard queue is full (backpressure), API returns a throttling response (e.g., 429) or offers a queued ticket.

Background workers: Snapshot serializer, journal flusher, and metrics exporter run asynchronously and do not block matching.

3. Matching Data Structures & Algorithms

Price levels: TreeMap<price, PriceLevel> (ascending/descending views as needed). A PriceLevel contains a FIFO queue of orders.

Use O(log N) access to best price and O(1) enqueue/dequeue per price level.

Matching loop:

Lockless read of incoming order

While not fully filled and opposite book non-empty:

take best price level

match as much as possible (partial fills supported)

append trade event to WAL and publish to NATS

If remainder > 0 and limit order, insert into price level queue

4. Recovery Strategy

WAL-first: All order arrivals and matched trades are appended synchronously to WAL (Postgres) as an append-only row. A coarse-grained durability setting allows batching of WAL writes for latency vs durability trade-off.

Periodic snapshots: Every X seconds or Y events, the shard writes a snapshot of its in-memory book (compact representation: price→aggregate quantity + queue positions hashed).

Fast restart:

Load latest snapshot if available.

Replay WAL entries after snapshot timestamp to bring state current.

Resume processing new journal entries.

Idempotency: Each order carries clientOrderId and server-assigned orderId; WAL deduplication ensures safe replays.

5. Trade-offs & Rationale

Single-writer shard vs fine-grained locks: Single-writer simplifies correctness and avoids lock contention. Trade-off: hot symbols can overload a shard — mitigated via auto-splitting or routing by instrument-categories in future.

Synchronous WAL vs async append: Synchronous WAL ensures durability but adds latency; we allow configurable durability (sync per event vs batched fsync).

Snapshots frequency: Frequent snapshots reduce replay time but cost IO and storage — choose a balanced window (e.g., every 10k events or 30s).

NATS vs Kafka: NATS offers low latency and simple ops (chosen for default); Kafka can be swapped in if longer retention and consumer replay are critical.

API Endpoints — curl examples + Postman collection
Major endpoints (examples)
Place an order
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "clientOrderId":"my-123",
    "symbol":"BTCUSDT",
    "side":"BUY",
    "type":"LIMIT",
    "price":"70050.00",
    "quantity":"0.001"
  }'


Example response:

{
  "orderId": "c8f2d8b1-...",
  "clientOrderId": "my-123",
  "status": "ACCEPTED",
  "timestamp": 1699990000000
}

Cancel an order
curl -X POST http://localhost:8080/api/v1/orders/cancel \
  -H "Content-Type: application/json" \
  -d '{"orderId":"c8f2d8b1-..."}'

Get order status
curl http://localhost:8080/api/v1/orders/c8f2d8b1-...

Get light snapshot of orderbook
curl http://localhost:8080/api/v1/book/BTCUSDT

WebSocket subscription (example)

Connect to ws://localhost:8080/ws and send:

{
  "type": "subscribe",
  "channels": ["book.BTCUSDT", "trades.BTCUSDT"]
}

Postman collection (importable JSON)

Save the JSON below to postman_collection.json and import into Postman:

{
  "info": {
    "name": "Matching Engine API",
    "_postman_id": "a1b2c3d4-xxxx",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Place Order",
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type", "value": "application/json" }
        ],
        "url": { "raw": "http://localhost:8080/api/v1/orders", "host": ["http://localhost:8080"], "path": ["api","v1","orders"] },
        "body": {
          "mode": "raw",
          "raw": "{\n  \"clientOrderId\":\"my-123\",\n  \"symbol\":\"BTCUSDT\",\n  \"side\":\"BUY\",\n  \"type\":\"LIMIT\",\n  \"price\":\"70050.00\",\n  \"quantity\":\"0.001\"\n}"
        }
      }
    },
    {
      "name": "Cancel Order",
      "request": {
        "method": "POST",
        "header": [{ "key": "Content-Type", "value": "application/json" }],
        "url": { "raw": "http://localhost:8080/api/v1/orders/cancel", "host": ["http://localhost:8080"], "path": ["api","v1","orders","cancel"] },
        "body": {
          "mode": "raw",
          "raw": "{ \"orderId\":\"<orderId>\" }"
        }
      }
    },
    {
      "name": "Get Book",
      "request": {
        "method": "GET",
        "url": { "raw": "http://localhost:8080/api/v1/book/BTCUSDT", "host": ["http://localhost:8080"], "path": ["api","v1","book","BTCUSDT"] }
      }
    }
  ]
}

Short Load Test Report & Scaling Plan (1 page)
Sample load-test setup (how we tested)

Environment: Local cluster using Docker Compose with MATCHER_SHARD_COUNT=4.

Load tool: k6 script loadtests/k6_submit_orders.js (200 VUs ramped for 2 minutes).

Test target: POST /api/v1/orders with randomized price/quantity for BTCUSDT.

Important: Below are sample results intended as a template to include in your submission. Replace them with your actual measured numbers from your runs.

Sample results (replace with your measured data)

Duration: 2 minutes (steady-state)

Aggregate throughput: ~2,400 orders/sec sustained (peak 2,650)

Median latency (server side): 1.8 ms

p95 latency: 6.4 ms

p99 latency: 12.1 ms

Error rate: 0.0% (no 5xx responses)

Matcher node CPU (single node observed): 75% at peak, Memory: 1.2 GB

WAL write latency median: 0.4 ms

Interpretation: With 4 shards, the system sustained >2k orders/sec on a developer host. Real numbers will vary by instance type and disk performance (fast NVMe recommended).

How to reproduce

Start full stack: docker compose up --build.

Run k6 as documented in [Run Tests].

Monitor metrics at /metrics (Prometheus) and check logs.

Scaling to multi-node / multi-instrument production
Horizontal scaling (multi-node)

Shard partitioning: Deploy matcher shard replicas across multiple nodes; each replica owns a set of partitions (symbols). Use a partition manager / consistent hashing for rebalancing.

Service discovery: Use Kubernetes with StatefulSets and headless services or a simple registry for shard assignment.

Autoscaling: Monitor queue depth and per-shard CPU; auto-scale replica counts when backlog > threshold.

WAL & persistence:

Prefer a strongly-consistent DB (Postgres / CockroachDB). For high throughput, shard ledger by shard-id (table per shard) to avoid write hotspots.

Use asynchronous batched WAL flush with commit logs replicated to standby for durability.

Streaming & consumers:

Use NATS JetStream (or Kafka) with topics per symbol for consumer scalability; consumers can replay deltas if they fall behind.

Load balancing & routing:

Central ingress routes incoming orders by symbol hash to the appropriate shard worker (or sticky connection via WebSocket).

Optionally have a lightweight gateway per region to reduce cross-region latency.

Multi-instrument / hot-symbol handling

Auto-split hot symbols: Monitor per-symbol throughput. For hot symbols, split the book into price ranges (e.g., high/low) or use multi-level partitioning.

Order routing policy: For client orders, use a router that routes to the shard owning the symbol's partition. Implement circuit breakers and spillover mechanisms.

Stateful migration: Implement snapshot-and-swap: freeze incoming matching for minutes, snapshot book, migrate snapshot to another node, replay WAL, switch routing.

Other production considerations

Persistence & backups: Regular snapshot to S3 + WAL retention policy; cross-region replication for DR.

Security: mTLS, HMAC-signed requests, strong auth for websockets, and WAF in front of API.

Observability: Prometheus/Grafana dashboards, alerting (backlog, p99 latency), distributed tracing (OpenTelemetry).

Testing: Production-grade chaos tests (network partition, disk slowdowns), replay-based deterministic regression tests.

Appendix: Useful commands & tips

Start dev stack: docker compose up --build

Stop & remove: docker compose down -v

Run unit tests: go test ./... -v

Build matcher binary: go build -o matcher ./cmd/matcher

Run k6: k6 run loadtests/k6_submit_orders.js

Check NATS monitoring UI: http://localhost:8222 (if enabled)

Prometheus/Grafana: add container or use hosted stack for visual metrics
