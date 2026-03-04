# Real-Time GPS-Guided Patrol Routing System (AWS MVP)

## 1) High-level architecture

The MVP is designed as an event-driven microservice architecture on AWS with four core planes:

1. **Ingestion plane** receives GPS updates from patrol vehicles in real time.
2. **Optimization plane** computes and updates patrol routes using zone, hotspot, and live-position data.
3. **Coordination plane** enforces non-overlap and spatial coverage policies.
4. **Officer delivery plane** pushes route updates to mobile clients via low-latency APIs.

### Core AWS services

- **Amazon API Gateway (HTTP + WebSocket APIs)**: secure mobile-facing entrypoint and push channel.
- **AWS IoT Core (MQTT)**: real-time telemetry ingest from vehicle devices.
- **Amazon Kinesis Data Streams**: scalable position-event backbone.
- **AWS Lambda / Amazon ECS (Fargate)**: stream processing, assignment, and routing microservices.
- **Amazon Aurora PostgreSQL + PostGIS**: transactional source of truth for officers, assignments, zones, and geospatial queries.
- **Amazon ElastiCache (Redis)**: low-latency state for active sessions, locks, and quick nearest-neighbor lookups.
- **Amazon OpenSearch Service** (optional for MVP, useful early): hotspot scoring, spatiotemporal analytics/search.
- **Amazon S3 + AWS Glue + Athena**: historical telemetry lake and offline analytics.
- **Amazon EventBridge**: domain events and integration fan-out.
- **Amazon Cognito**: identity/auth for officer mobile apps.
- **AWS AppConfig / Parameter Store / Secrets Manager**: feature flags and secure configuration.
- **Amazon CloudWatch + AWS X-Ray + AWS Managed Prometheus/Grafana**: observability.

---

## 2) Component-by-component design

## (1) Real-time GPS ingestion backend

### Device connectivity

- Patrol vehicles run a lightweight edge client publishing GPS payloads every 1–5 seconds over MQTT to **AWS IoT Core**.
- Payload example:

```json
{
  "vehicleId": "V-1027",
  "officerId": "O-447",
  "ts": "2026-03-04T12:00:05Z",
  "lat": 40.74218,
  "lon": -73.99122,
  "speedMps": 12.1,
  "headingDeg": 148,
  "accuracyM": 4.8,
  "status": "on_patrol"
}
```

### Ingestion flow

1. IoT Core authenticates device certificate and validates topic permissions.
2. IoT Rule forwards telemetry to **Kinesis Data Streams** (`gps-position-stream`).
3. A **Stream Processor** (Lambda for early MVP; ECS/Flink if volume grows) performs:
   - schema validation,
   - deduplication (vehicleId + timestamp + geohash key),
   - map matching / smoothing (optional),
   - event enrichment (zoneId lookup via PostGIS/Redis),
   - persistence of latest location in Redis and append in Aurora/S3.

### Storage pattern

- **Redis**: latest known position per vehicle/officer (`SETEX position:{officerId}`).
- **Aurora PostGIS**: authoritative event table (partitioned by day/week) for operational queries.
- **S3**: compressed raw telemetry for replay and model training.

---

## (2) Routing engine for optimized patrol routes

### Routing service responsibilities

- Inputs:
  - current officer positions,
  - patrol zones,
  - hotspot intensity map,
  - shift constraints (time, priority calls, restricted roads),
  - fairness/load constraints.
- Outputs:
  - route plans with ordered waypoints,
  - ETA and expected coverage score.

### Implementation choice

- **Routing Engine microservice** on ECS Fargate (Python or Rust/Go).
- For road-network travel times:
  - Start with **OSRM/GraphHopper** hosted in ECS (city-region extracts in EBS/S3).
  - Alternative managed option: Amazon Location Service route calculations (faster MVP if acceptable pricing/features).
- Optimization approach:
  - MVP heuristic: weighted coverage maximization + tabu/local search for near-real-time response.
  - Re-plan triggers: major position drift, hotspot updates, officer unavailable, incident priority change.

### Data used

- **Hotspots**: computed by Hotspot Service (see below), stored as grid cells/geohashes with decaying scores in Aurora/OpenSearch.
- **Zones**: PostGIS polygons for beats/sectors with policy metadata.

---

## (3) Coordination layer (non-overlap + coverage assurance)

### Coordination service

A dedicated **Coverage Coordinator** service continuously enforces constraints:

- No two officers should patrol the same micro-cell at same time unless explicitly allowed.
- Coverage SLA: each zone must meet minimum revisit interval.
- Dynamic balancing: if one zone under-covered, reassign nearest eligible unit.

### Technical mechanics

- Represents city as hierarchical spatial index (H3 or geohash).
- Maintains occupancy and “planned occupancy horizon” in Redis:
  - `occupancy:{cell}:{timeslice}` -> officerId
  - distributed lock with short TTL to avoid race conditions during assignment.
- Persists final assignments in Aurora (transactional integrity, auditing).
- Emits domain events via EventBridge/Kinesis:
  - `RouteAssigned`, `RouteReassigned`, `CoverageGapDetected`, `OverlapConflictResolved`.

### Conflict resolution strategy

1. Detect overlap in planned route segments.
2. Score alternatives by added travel time + coverage gain.
3. Apply best reassignment atomically (Redis lock + Aurora transaction).
4. Push updated route to affected officers.

---

## (4) Mobile-facing API for push route updates

### API surface

Use two channels:

1. **REST (API Gateway HTTP API)** for command/query operations.
2. **WebSocket (API Gateway WebSocket API)** for near real-time route updates (<2s target).

### Core endpoints (REST)

- `POST /v1/officer/session/start`
- `POST /v1/officer/session/heartbeat`
- `GET /v1/officer/route/current`
- `POST /v1/officer/route/ack`
- `POST /v1/officer/status` (available, busy, enroute_incident)
- `GET /v1/officer/hotspots?bbox=...`

### WebSocket events

- Server -> client:
  - `route.updated`
  - `route.cancelled`
  - `coverage.alert`
- Client -> server:
  - `route.received`
  - `route.progress`

### Push reliability

- Every route update has `routeVersion` and `correlationId`.
- Mobile app acks receipt; unacked updates retried with exponential backoff.
- If websocket disconnected, fallback to push notification + REST pull on reconnect.

---

## 3) API contract sketch

## Route object

```json
{
  "routeId": "R-20260304-00921",
  "routeVersion": 7,
  "officerId": "O-447",
  "zoneId": "Z-12",
  "priority": "normal",
  "validFrom": "2026-03-04T12:00:20Z",
  "validTo": "2026-03-04T12:30:20Z",
  "waypoints": [
    {"seq": 1, "lat": 40.7428, "lon": -73.9921, "eta": "2026-03-04T12:03:10Z"},
    {"seq": 2, "lat": 40.7452, "lon": -73.9874, "eta": "2026-03-04T12:07:50Z"}
  ],
  "coverageCells": ["8a2a1072b59ffff", "8a2a1072b4dffff"],
  "reason": "hotspot_rebalance"
}
```

## Idempotency and consistency

- REST write endpoints require `Idempotency-Key` header.
- Use optimistic concurrency on route acknowledgements (`routeVersion` check).
- Exactly-once is approximated via dedupe keys and idempotent handlers.

---

## 4) Data model and database choices

## Operational relational store: Aurora PostgreSQL + PostGIS

Tables (MVP core):

- `officers`
- `vehicles`
- `patrol_zones` (geometry polygon)
- `gps_events` (time-partitioned)
- `hotspot_cells` (cell_id, score, updated_at)
- `route_plans`
- `route_waypoints`
- `assignments`
- `coverage_audit`

PostGIS enables:

- point-in-polygon zone assignment,
- nearest patrol unit queries,
- overlap/intersection checks,
- geospatial indexing (GiST/SP-GiST).

## In-memory state: Redis

- Active sessions, presence, locks, and latest locations.
- TTL-based ephemeral state to keep coordinator fast.

## Analytics/search store: OpenSearch (optional but useful)

- Spatiotemporal hotspot exploration,
- operational dashboards for supervisors.

## Data lake: S3

- Raw and curated telemetry for long-term analytics, compliance, and ML.

---

## 5) Service-to-service communication pattern

## Synchronous paths

- Mobile app -> API Gateway -> Officer API service (ECS/Lambda).
- Officer API -> Aurora/Redis for low-latency reads/writes.

## Asynchronous/event paths

- GPS events: IoT Core -> Kinesis -> Stream Processor -> EventBridge/Kinesis.
- Route lifecycle: Routing Engine/Coordinator -> EventBridge (`RouteAssigned`, etc.).
- Notification worker subscribes and pushes via WebSocket management API.

## Why this split

- Synchronous where user-facing latency matters.
- Asynchronous where decoupling, retries, and burst tolerance matter.

---

## 6) Hotspot pipeline

1. GPS + incident + historical crime feeds land in Kinesis/S3.
2. Hotspot service computes decayed risk score per H3/geohash cell every N minutes.
3. Writes latest scores to Aurora (`hotspot_cells`) and OpenSearch.
4. Emits `HotspotMapUpdated` event; routing engine re-optimizes impacted zones.

MVP algorithm:

- Weighted moving average of recent incidents + patrol visibility deficit.
- Time-of-day seasonal multiplier.
- Optional weather/event multipliers later.

---

## 7) Security, compliance, and governance

- **AuthN/AuthZ**: Cognito JWT for mobile users, IAM roles for services, mTLS/certs for vehicle devices.
- **Encryption**: TLS in transit; KMS encryption at rest across Aurora, Kinesis, S3, OpenSearch.
- **Auditability**: immutable route assignment audit logs in S3 + CloudTrail.
- **Least privilege**: scoped IAM per microservice.
- **PII minimization**: avoid storing unnecessary personal data in telemetry payloads.

---

## 8) Availability, scaling, and SLOs

Target MVP SLOs:

- GPS ingest availability: 99.9%
- Position-to-route-update p95 latency: < 5s
- Route push success (with retries): > 99%

Scaling levers:

- Kinesis shard scaling by ingestion throughput.
- ECS service autoscaling on CPU + queue lag.
- Read replicas in Aurora for heavy read patterns.
- Redis clustering when active officer count rises.

Multi-AZ baseline for all stateful services.

---

## 9) Suggested MVP deployment topology (AWS)

- **VPC** across 3 AZs.
- Private subnets: ECS services, Aurora, Redis, OpenSearch.
- Public ingress only through API Gateway and IoT Core.
- NAT gateway for controlled outbound access.
- CI/CD via GitHub Actions or CodePipeline -> ECR -> ECS deploy.

Environments:

- `dev` (single-AZ cost-optimized),
- `staging` (prod-like, integration tests),
- `prod` (multi-AZ, autoscaling enabled).

---

## 10) MVP implementation roadmap

Phase 1 (Weeks 1–3):

- GPS ingest pipeline (IoT -> Kinesis -> processor -> Aurora/Redis).
- Mobile auth/session APIs.
- Basic map UI consuming current route endpoint.

Phase 2 (Weeks 4–6):

- Routing engine v1 with static hotspot inputs.
- Coordinator with overlap prevention and reassignment.
- WebSocket push route updates.

Phase 3 (Weeks 7–8):

- Hotspot scoring pipeline.
- SLO dashboards + alerting.
- Replay and backtesting on historical data.

---

## 11) Minimal initial tech stack recommendation

- **Language**: Python (fast iteration) for ingest/coordinator/routing; Go for high-throughput components if needed.
- **Frameworks**: FastAPI (Officer API), asyncio workers, SQLAlchemy.
- **Infra as code**: AWS CDK (TypeScript/Python) or Terraform.
- **Data**: Aurora Postgres + PostGIS, Redis, S3, optional OpenSearch.
- **Messaging/stream**: IoT Core + Kinesis + EventBridge.

This gives fast MVP delivery while preserving an easy growth path to higher scale and stricter operational requirements.
