# RideShare# Ride-Sharing Service (Uber Clone) — Complete System Design Blueprint

---

## 1️⃣ Problem Understanding

### Clarification

We're designing a platform that connects riders needing transportation with nearby drivers in real time. The system must handle location tracking, dynamic pricing, matching, payments, and trip management at massive scale across geographically distributed regions.

### Assumptions

- Mobile-first (iOS/Android) with web support for admin/ops
- Operates in multiple cities, potentially globally
- Drivers use a dedicated driver app; riders use a rider app
- Real-time GPS updates every 3–5 seconds from active drivers
- Dynamic surge pricing based on supply/demand
- Payment is handled via integration with payment gateways (Stripe, Braintree)
- Regulatory compliance varies by region (GDPR in EU, CCPA in US)

### Scope

**Included:** Rider/driver onboarding, ride request & matching, real-time tracking, fare calculation, payments, ratings, trip history, surge pricing, notifications, basic analytics.

**Excluded:** Carpooling/pooled rides, food delivery, fleet management, driver background check pipeline internals, insurance claims, in-app chat.

---

## 2️⃣ Functional Requirements

### Core Features

**Rider side:** Register/login, set pickup & destination, see fare estimate, request ride, track driver in real-time, pay automatically, rate driver, view trip history.

**Driver side:** Register/login, toggle availability, receive ride requests, accept/decline, navigate to rider, complete trip, receive payout, view earnings.

**Platform side:** Matching engine, surge pricing, fraud detection, dispatch, notifications.

### User Flows

**Happy path — Rider:** Open app → GPS locates rider → Enter destination → See ETA + fare estimate → Tap "Request Ride" → Matched to driver → Track driver approach → Driver arrives → Trip begins → Trip ends → Auto-charge → Rate driver.

**Happy path — Driver:** Go online → Receive ping → Accept within 10s → Navigate to pickup → Tap "Arrived" → Rider gets in, tap "Start Trip" → Navigate to destination → Tap "End Trip" → Fare calculated → Payout queued.

### Edge Cases

- Driver cancels after acceptance → re-match rider, apply cancellation penalty
- Rider cancels after driver en route → partial cancellation fee after threshold
- GPS drift causing incorrect fare → trip correction flow
- No drivers available → rider placed in queue, surge pricing activated
- Payment failure → retry logic, fallback to stored methods, ride debt tracking
- Driver goes offline mid-trip → trip continues, alert ops team
- Surge pricing disagreement → fare lock at booking time
- Simultaneous ride requests flooding the same driver

---

## 3️⃣ Non-Functional Requirements

### Scalability Targets

- 50M daily active riders, 5M daily active drivers globally
- Peak: 500K concurrent ride requests
- Location updates: ~5M GPS pings/second at peak
- Matching latency: < 500ms P99
- API gateway: 2M+ RPS at peak

### Latency Expectations

- Ride match: < 2s end-to-end
- Driver location update ingestion: < 100ms
- Payment processing: < 3s
- Fare estimate: < 300ms
- Push notification delivery: < 1s

### Availability

- Core ride flow: 99.99% SLA (52 min downtime/year)
- Payments: 99.999%
- Driver location tracking: 99.95%
- Trip history / non-critical: 99.9%

### Consistency

- Strong consistency: payments, ride state machine
- Eventual consistency: driver location, ratings, analytics
- Causal consistency: ride matching (no double-assignment)

### Security

- PII encryption, PCI-DSS for payments, OAuth 2.0 + JWT, TLS 1.3 everywhere, rate limiting, fraud detection ML pipeline

### Compliance

- GDPR (right to erasure, data residency), CCPA, PCI-DSS Level 1, local taxi/transport regulations per market

---

## 4️⃣ Capacity Estimation

### Users & Traffic

```
Daily Active Riders:        50M
Daily Active Drivers:        5M
Avg trips/rider/day:         1.5
Total trips/day:            ~75M trips/day
Peak trips/second:          75M / 86400 * 3 (peak factor) ≈ 2,600 TPS
Ride requests/second:       ~5,000 RPS at peak (including retries, surges)
```

### Location Updates

```
Active drivers at peak:     500K
GPS ping every 4 seconds:   500K / 4 = 125,000 location updates/sec
Per update payload:         ~200 bytes
Bandwidth (ingress):        125K * 200B = 25 MB/s ingress
```

### Storage

```
Trip record size:           ~2KB
Trips/day:                  75M
Daily trip storage:         75M * 2KB = 150 GB/day
Annual (compressed):        ~20 TB/year
Location history (30 days): 125K updates/sec * 200B * 86400 * 30 ≈ 25 TB
User records (100M total):  ~100 GB
```

### Peak vs Average

|Metric|Average|Peak (3×)|
|---|---|---|
|Ride requests/sec|870|2,600|
|Location updates/sec|40K|125K|
|API gateway RPS|500K|2M|
|Payment TPS|870|2,600|

---

## 5️⃣ High-Level Architecture

### Microservices Decision

**Decision: Microservices with domain-driven boundaries.**

Reasoning: The system has clearly independent domains (location, matching, payments, trips) with vastly different scaling profiles. Location service needs to handle 125K writes/sec while user service is mostly read-heavy. Independent deployability is critical for a system with 99.99% SLA — a monolith would make rolling updates risky. Teams can be organized around services (Conway's Law advantage). The complexity cost of microservices is justified at this scale.

### Major Services

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway / BFF                     │
│              (Kong / AWS API Gateway)                    │
└────────┬──────────┬──────────┬──────────┬───────────────┘
         │          │          │          │
    ┌────▼──┐  ┌────▼──┐  ┌───▼───┐  ┌──▼──────┐
    │ Auth  │  │ User  │  │ Trip  │  │Location │
    │Service│  │Service│  │Service│  │ Service │
    └───────┘  └───────┘  └───────┘  └─────────┘
                               │
              ┌────────────────┼────────────────┐
         ┌────▼────┐    ┌─────▼────┐    ┌──────▼─────┐
         │Matching │    │Pricing/  │    │ Payment    │
         │ Engine  │    │Surge Svc │    │ Service    │
         └─────────┘    └──────────┘    └────────────┘
                               │
              ┌────────────────┼────────────────┐
         ┌────▼────┐    ┌─────▼────┐    ┌──────▼─────┐
         │Notif.   │    │Analytics │    │ Rating     │
         │ Service │    │ Service  │    │ Service    │
         └─────────┘    └──────────┘    └────────────┘

Shared Infrastructure:
- Kafka (event bus)
- Redis Cluster (caching, geo-index, pub/sub)
- PostgreSQL clusters (OLTP)
- Cassandra (location history, time-series)
- S3 (documents, receipts)
- Elasticsearch (search, analytics)
```

### Communication Patterns

- **Synchronous (gRPC):** Auth↔Trip, Trip↔Payment, Trip↔Matching (latency-critical)
- **Asynchronous (Kafka):** Trip events → Notifications, Analytics, Rating prompts
- **WebSocket / SSE:** Real-time driver location to rider app
- **REST:** External-facing APIs via API gateway

---

## 6️⃣ Detailed Component Design

### 6.1 Auth Service

**Responsibilities:** Registration, login, JWT issuance, token refresh, OAuth (Google/Apple sign-in), driver credential verification.

**APIs:**

```
POST /auth/register          → { userId, token, refreshToken }
POST /auth/login             → { token, refreshToken }
POST /auth/refresh           → { token }
POST /auth/logout
POST /auth/verify-driver     → { driverId, status }
```

**Data Model:**

```sql
users (
  id          UUID PRIMARY KEY,
  phone       VARCHAR(20) UNIQUE NOT NULL,
  email       VARCHAR(255) UNIQUE,
  password_hash VARCHAR(255),
  role        ENUM('rider','driver','admin'),
  created_at  TIMESTAMP,
  verified_at TIMESTAMP
)

refresh_tokens (
  token_hash  VARCHAR(64) PRIMARY KEY,
  user_id     UUID REFERENCES users(id),
  expires_at  TIMESTAMP,
  revoked     BOOLEAN DEFAULT FALSE
)
```

**DB Choice:** PostgreSQL — ACID compliance needed, relatively low write volume, complex joins for credential management.

**Caching:** JWT public keys cached in Redis (5min TTL). Blacklisted tokens in Redis SET.

**Token Strategy:** Short-lived JWT (15min access token) + long-lived refresh token (30 days). Refresh tokens stored server-side for revocability. RS256 signing.

---

### 6.2 Location Service

**Responsibilities:** Ingest driver GPS pings, maintain current driver locations in geospatial index, serve nearby driver queries to matching engine, stream location to rider apps.

**APIs:**

```
# gRPC (high throughput)
rpc UpdateLocation(LocationUpdate) returns (Ack)
rpc GetNearbyDrivers(NearbyRequest) returns (DriverList)
rpc StreamDriverLocation(DriverId) returns (stream LocationUpdate)

# REST (external)
GET /location/drivers/nearby?lat=&lng=&radius=&limit=
```

**Data Model:**

```
LocationUpdate {
  driver_id: string
  lat: double
  lng: double
  heading: float
  speed: float
  timestamp: int64
  status: enum (AVAILABLE, ON_TRIP, OFFLINE)
}
```

**Storage Architecture:**

```
Ingest path:
  Driver App → gRPC → Location Service → Redis GEO + Kafka

Redis GEO index:
  GEOADD drivers:available <lng> <lat> <driver_id>
  GEORADIUS drivers:available <lng> <lat> 5 km ASC COUNT 20

Kafka Topic: driver-location-updates
  → Consumer: Location History Writer → Cassandra
  → Consumer: Rider Location Streamer → WebSocket push

Cassandra (location history):
  driver_locations_by_driver (
    driver_id UUID,
    ts        TIMESTAMP,
    lat       DOUBLE,
    lng       DOUBLE,
    PRIMARY KEY (driver_id, ts)
  ) WITH CLUSTERING ORDER BY (ts DESC)
```

**DB Choice:** Redis for current state (sub-millisecond geo queries), Cassandra for history (time-series, high write throughput, TTL).

**Throughput Design:** 125K writes/sec → 50 Location Service pods, each handling ~2,500 writes/sec. Redis Cluster with 20 shards.

**WebSocket Streaming:** Rider subscribes to `ws://location-service/driver/{driverId}`. Location service maintains a pub/sub channel per active driver in Redis. Each location update publishes to channel; all subscribed rider WebSocket connections receive it.

---

### 6.3 Matching Engine

**Responsibilities:** Accept ride requests, find best available driver, handle acceptance/rejection, manage the matching state machine, implement ETA-based ranking.

**Matching Algorithm:**

```
1. Receive ride request with pickup coordinates
2. Query Location Service: drivers within 5km radius, sorted by distance
3. Filter: available status, vehicle type match, not blacklisted
4. Rank by: weighted score = 0.5*distance + 0.3*ETA + 0.2*rating
5. Send offer to top driver (10s timeout)
6. If rejected/timeout → next driver
7. If no match in N attempts → expand radius, notify rider
8. On acceptance → atomically update driver status, create trip record
```

**APIs:**

```
POST /match/request
Body: { rider_id, pickup_lat, pickup_lng, dest_lat, dest_lng, vehicle_type }
Response: { request_id, status: "searching" }

POST /match/respond
Body: { driver_id, request_id, accepted: bool }
Response: { trip_id? }

GET /match/status/{request_id}
Response: { status, driver_id?, eta? }
```

**Concurrency — The Double-Assignment Problem:**

This is the critical race condition: two ride requests simultaneously sent to the same driver. Solution:

```
1. Driver offer = Redis SET driver:{id}:offered {request_id} EX 10 NX
   - NX = only set if not exists
   - If SET returns nil → driver already offered to another request
   - Move to next driver

2. On acceptance → Lua script (atomic):
   local current = redis.call('GET', KEYS[1])
   if current == ARGV[1] then
     redis.call('SET', KEYS[1], 'assigned')
     return 1
   end
   return 0

3. If script returns 0 → race lost, proceed to next driver
```

**State Machine:**

```
PENDING → SEARCHING → DRIVER_FOUND → DRIVER_EN_ROUTE
→ DRIVER_ARRIVED → IN_PROGRESS → COMPLETED | CANCELLED
```

Stored in Redis for active trips (fast access) + replicated to PostgreSQL for durability.

---

### 6.4 Trip Service

**Responsibilities:** Own the trip lifecycle state machine, coordinate between matching, payment, and notification services, persist trip records.

**APIs:**

```
POST /trips                  → Create trip record after match
GET  /trips/{tripId}         → Trip details
POST /trips/{tripId}/start   → Driver starts trip
POST /trips/{tripId}/end     → Driver ends trip → triggers billing
GET  /trips/user/{userId}    → Trip history
POST /trips/{tripId}/cancel  → Cancel with reason
```

**Data Model:**

```sql
trips (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rider_id        UUID NOT NULL,
  driver_id       UUID NOT NULL,
  status          VARCHAR(20) NOT NULL,
  pickup_lat      DECIMAL(9,6),
  pickup_lng      DECIMAL(9,6),
  pickup_address  TEXT,
  dest_lat        DECIMAL(9,6),
  dest_lng        DECIMAL(9,6),
  dest_address    TEXT,
  vehicle_type    VARCHAR(20),
  fare_estimate   DECIMAL(10,2),
  final_fare      DECIMAL(10,2),
  surge_multiplier DECIMAL(4,2) DEFAULT 1.0,
  requested_at    TIMESTAMP NOT NULL,
  matched_at      TIMESTAMP,
  started_at      TIMESTAMP,
  ended_at        TIMESTAMP,
  cancelled_at    TIMESTAMP,
  cancel_reason   TEXT,
  payment_id      UUID,
  route_encoded   TEXT,            -- polyline
  distance_km     DECIMAL(8,3),
  duration_secs   INTEGER
)

-- Partitioned by month for query performance
-- Indexed on: rider_id, driver_id, status, requested_at
```

**DB Choice:** PostgreSQL with table partitioning (by month). Read replicas for trip history queries. Archive completed trips > 90 days to S3 via partitioning + pg_partman.

**Event Publishing:** Every state change publishes to Kafka topic `trip-events`:

```json
{ "event": "TRIP_STARTED", "trip_id": "...", "ts": "...", "payload": {...} }
```

Consumers: Notification Service, Analytics, Payment Service (on TRIP_ENDED).

---

### 6.5 Pricing / Surge Service

**Responsibilities:** Calculate base fare, apply surge multiplier, provide fare estimates, expose surge data to front-end.

**Fare Formula:**

```
base_fare = BASE_RATE + (distance_km * PER_KM_RATE) + (duration_min * PER_MIN_RATE)
final_fare = max(MINIMUM_FARE, base_fare * surge_multiplier)
```

**Surge Calculation:**

```python
# Runs every 30 seconds per geographic hexagon (H3 index, resolution 7)
def calculate_surge(hex_id: str) -> float:
    demand = pending_requests_in_hex(hex_id, last_5_min)
    supply = available_drivers_in_hex(hex_id)
    ratio = demand / max(supply, 1)
    
    # Surge tiers
    if ratio < 1.0: return 1.0
    if ratio < 1.5: return 1.2
    if ratio < 2.0: return 1.5
    if ratio < 3.0: return 2.0
    return min(ratio * 0.8, 3.0)  # cap at 3x
```

Surge multipliers stored in Redis with 60s TTL, keyed by H3 hex ID.

**H3 Geospatial Indexing:** Uber's own H3 library divides the globe into hexagonal cells. Resolution 7 ≈ 5km² per cell. Drivers and requests mapped to hex IDs for O(1) lookup.

---

### 6.6 Payment Service

**Responsibilities:** Charge rider on trip completion, hold/pre-authorize card, handle refunds, manage driver payouts, fraud detection hooks.

**APIs:**

```
POST /payments/authorize          → Pre-auth card at ride start
POST /payments/capture/{tripId}   → Charge final amount at trip end
POST /payments/refund/{paymentId} → Full or partial refund
GET  /payments/methods/{userId}   → Saved payment methods
POST /payments/methods            → Add payment method
GET  /payments/history/{userId}
```

**Flow:**

```
Trip Requested → Pre-auth for estimated fare * 1.3 (surge buffer)
Trip Ended    → Capture actual amount
              → Release pre-auth delta
              → Queue driver payout (daily batch or instant)
Payment Fail  → Retry 3x with exponential backoff
              → Flag account, restrict future rides
              → Send recovery notification
```

**Data Model:**

```sql
payments (
  id              UUID PRIMARY KEY,
  trip_id         UUID UNIQUE,
  rider_id        UUID,
  driver_id       UUID,
  amount_cents    INTEGER NOT NULL,
  currency        CHAR(3),
  status          VARCHAR(20),  -- PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED
  gateway_txn_id  VARCHAR(100),
  gateway         VARCHAR(20),  -- stripe, braintree
  created_at      TIMESTAMP,
  updated_at      TIMESTAMP
)

payment_methods (
  id              UUID PRIMARY KEY,
  user_id         UUID,
  gateway_token   VARCHAR(255),  -- tokenized, never raw card data
  type            VARCHAR(20),   -- card, wallet
  last_four       CHAR(4),
  exp_month       SMALLINT,
  exp_year        SMALLINT,
  is_default      BOOLEAN
)
```

**Idempotency:** All payment operations include `Idempotency-Key: {trip_id}:{operation}`. Stored in Redis (24h TTL) — duplicate requests return cached response without re-charging.

**PCI Compliance:** Card data never touches our servers. Tokenization via Stripe.js/Braintree SDK on client side. We store only gateway tokens.

---

### 6.7 Notification Service

**Responsibilities:** Push notifications (FCM/APNs), SMS (Twilio), in-app real-time messages.

Consumes Kafka `trip-events` and `system-events` topics. Maps event types to notification templates. Handles device token management, delivery receipts, retry on failure.

```
Kafka Consumer → Template Resolver → FCM/APNs/Twilio → Delivery Tracking
```

Notifications are fire-and-forget with at-least-once delivery. Idempotency via deduplication window in Redis.

---

## 7️⃣ Data Modeling

### Entity Relationships

```
Users (1) ──── (N) Trips (as rider)
Drivers (1) ─── (N) Trips (as driver)
Trips (1) ────── (1) Payment
Trips (1) ────── (N) LocationHistory
Trips (1) ────── (1) Rating (rider→driver)
Trips (1) ────── (1) Rating (driver→rider)
Users (1) ──── (N) PaymentMethods
Drivers (1) ─── (1) DriverProfile (license, vehicle, docs)
```

### Indexing Strategy

```sql
-- trips table
CREATE INDEX idx_trips_rider ON trips(rider_id, requested_at DESC);
CREATE INDEX idx_trips_driver ON trips(driver_id, requested_at DESC);
CREATE INDEX idx_trips_status ON trips(status) WHERE status NOT IN ('COMPLETED','CANCELLED');

-- payments table
CREATE INDEX idx_payments_rider ON payments(rider_id, created_at DESC);
CREATE UNIQUE INDEX idx_payments_trip ON payments(trip_id);

-- users table
CREATE UNIQUE INDEX idx_users_phone ON users(phone);
CREATE UNIQUE INDEX idx_users_email ON users(email) WHERE email IS NOT NULL;
```

### Sharding Strategy

**Trip Service:** Shard by `rider_id % N` for write distribution. However, driver queries need a separate index or cross-shard query (acceptable — driver queries are less frequent). Use Citus (PostgreSQL sharding extension) or migrate to Vitess.

**Location Service:** Shard Redis GEO by geographic region (continent/country). Matching engine connects to regional Redis cluster. No cross-region location lookup needed (a ride is always local).

**Cassandra (Location History):** Natural partitioning by `driver_id`. Data locality is not a concern — we query by driver, never by geography historically.

---

## 8️⃣ Real-Time & Concurrency Strategy

### Locking Strategy

|Scenario|Lock Type|Implementation|
|---|---|---|
|Driver assignment|Optimistic|Redis SET NX (see §6.3)|
|Payment capture|Pessimistic|DB row lock (`SELECT FOR UPDATE`)|
|Trip state transition|Optimistic|Version field + CAS|
|Surge calculation|None needed|Eventual consistency acceptable|

### Trip State CAS Pattern

```sql
UPDATE trips 
SET status = 'IN_PROGRESS', started_at = NOW(), version = version + 1
WHERE id = $1 AND status = 'DRIVER_ARRIVED' AND version = $2;
-- If 0 rows updated → concurrent modification detected → retry/reject
```

### Message Queue Strategy (Kafka)

**Topics and partitioning:**

```
driver-location-updates     → partitioned by driver_id (500 partitions)
trip-events                 → partitioned by trip_id (200 partitions)
payment-events              → partitioned by rider_id (100 partitions)
notification-jobs           → partitioned by user_id (100 partitions)
```

**Consumer groups:**

- `location-history-writer` — writes to Cassandra
- `rider-location-streamer` — pushes to WebSocket connections
- `analytics-collector` — writes to data warehouse
- `notification-dispatcher` — sends push/SMS

### Idempotency Handling

All Kafka consumers implement idempotent processing:

```python
def process_event(event):
    dedupe_key = f"processed:{event.topic}:{event.offset}"
    if redis.set(dedupe_key, 1, nx=True, ex=86400):
        # First time processing → execute
        handle_event(event)
    # else → already processed, skip silently
```

### WebSocket Architecture for Real-Time Tracking

```
Rider App → WebSocket → WS Gateway (sticky session) → Redis PubSub
Driver App → gRPC → Location Service → Redis PubSub → WS Gateway → Rider App

WS Gateway maintains:
  driver_id → [set of connected rider websocket connections]
  
On location update:
  1. Location Service: PUBLISH driver:{id}:location {lat,lng,heading}
  2. WS Gateway subscribes, receives message
  3. Forwards to all rider WebSocket connections watching that driver
```

Horizontal scaling of WS Gateway: sticky sessions via consistent hashing at load balancer. When a WS Gateway pod dies, clients reconnect (auto-reconnect with exponential backoff in SDK). Redis PubSub subscription is re-established on new pod.

---

## 9️⃣ Scalability & Performance Optimization

### Horizontal Scaling

Every service is stateless and horizontally scalable behind a load balancer. State lives in Redis/PostgreSQL/Cassandra — not in application memory. Kubernetes HPA scales pods based on CPU + custom metrics (queue depth, RPS).

### Load Balancing

```
Global: AWS Global Accelerator / Cloudflare (anycast routing to nearest region)
L7:     Kong API Gateway (per region) → service mesh (Istio)
Service mesh: gRPC load balancing with client-side load balancing (pick_first → round_robin)
WebSocket: L4 NLB with consistent hashing
```

### Caching Layers

```
L1: In-process cache (Caffeine/Guava)
    → Config values, surge multipliers (5s TTL, short-lived hot data)
    
L2: Redis Cluster
    → Driver locations (current state, geo index)
    → Active trip state
    → User session / JWT blacklist
    → Fare estimates (2min TTL, keyed by lat/lng grid cell)
    → Driver offer state
    
L3: CDN (CloudFront)
    → Static assets, map tiles
    → Driver documents (profile photos)
```

### Read Replicas

PostgreSQL: 1 primary + 2 read replicas per region. Trip history, user profile reads routed to replicas. Write latency shielded from read traffic.

### Partitioning Strategy

Trips table partitioned by month (PostgreSQL declarative partitioning). Queries always include `requested_at` range — partition pruning eliminates irrelevant partitions. Old partitions archived to S3 via pg_partman + custom archival job.

### Geographic Distribution

```
Regions: US-EAST, US-WEST, EU-WEST, AP-SOUTH, AP-EAST
Each region is fully independent (no cross-region real-time dependency)
Users assigned to nearest region at registration
Cross-region: only for analytics, fraud signals, global user lookup
```

---

## 🔟 Reliability & Fault Tolerance

### Circuit Breakers

Using Resilience4j (Java) or circuit-breaker-go (Go). Applied on every external service call:

```yaml
payment-gateway-breaker:
  failureRateThreshold: 50%
  waitDurationInOpenState: 30s
  slidingWindowSize: 10 calls

matching-engine-breaker:
  failureRateThreshold: 30%
  waitDurationInOpenState: 10s
```

**Fallback behaviors:**

- Payment gateway down → queue payment, retry async, allow trip to complete
- Matching engine degraded → fall back to distance-only matching (simplified)
- Location service degraded → use last known location (with timestamp warning)

### Retry Strategy

```
Exponential backoff with jitter:
  attempt 1: immediate
  attempt 2: 100ms + jitter
  attempt 3: 500ms + jitter
  attempt 4: 2s + jitter
  max_attempts: 4 for idempotent ops, 1 for non-idempotent (payments)
```

### Rate Limiting

API Gateway level (Kong + Redis):

```
Rider: 100 req/min general, 10 ride requests/min, 5 location updates/sec
Driver: 200 req/min general, 60 location updates/min (enforced client-side primarily)
Anonymous: 20 req/min
```

Token bucket algorithm per user_id + IP.

### Graceful Degradation

|Failure|Degraded Mode|
|---|---|
|Surge service down|Default 1.0x multiplier|
|Rating service down|Skip rating prompt, process async|
|Analytics pipeline lag|User-facing unaffected|
|Notification service down|Core trip flow unaffected|
|Location history down|Current location still available|

### Disaster Recovery

**RPO (Recovery Point Objective):** < 1 minute for trip data (WAL streaming replication) **RTO (Recovery Time Objective):** < 5 minutes for core ride flow

Strategy:

- PostgreSQL: streaming replication cross-AZ, daily snapshots to S3, point-in-time recovery
- Redis: AOF + RDB snapshots, Redis Sentinel / Redis Cluster with cross-AZ replicas
- Cassandra: Multi-datacenter replication (RF=3 per DC)
- Kafka: Cross-AZ replication, 7-day retention

Active-Active multi-region: each region serves its local users. Route 53 health checks + failover. If US-EAST degrades, us-east traffic reroutes to US-WEST (5% latency increase, acceptable).

---

## 1️⃣1️⃣ Security Design

### Authentication & Authorization

```
Auth flow:
1. User signs in → Auth Service issues JWT (RS256, 15min) + refresh token
2. JWT contains: user_id, role, region, device_id
3. API Gateway validates JWT signature using public key (cached)
4. Services receive pre-validated user context in headers
5. No inter-service JWT re-validation (service mesh mTLS ensures internal trust)

OAuth 2.0:
- Google/Apple Sign-In for riders
- Phone OTP (Twilio Verify) as primary driver authentication
```

**Authorization:** RBAC enforced at service level. Riders cannot call driver-only endpoints; enforced via role claim in JWT + API Gateway route policies.

### Encryption

- **In transit:** TLS 1.3 everywhere. gRPC over TLS. Service mesh (Istio) provides mTLS between all internal services automatically.
- **At rest:** AWS KMS-managed encryption for RDS, S3, EBS volumes. Application-level encryption for PII fields (AES-256-GCM) — phone numbers, email stored encrypted, encrypted index maintained for lookups.
- **Secrets:** AWS Secrets Manager / HashiCorp Vault. No secrets in environment variables or code.

### DDoS Protection

Cloudflare in front of all public endpoints. AWS Shield Advanced for infrastructure protection. Rate limiting at CDN edge before traffic reaches API Gateway. Behavioral analysis for bot detection (anomalous location update patterns, request fingerprinting).

### Data Protection

- Driver documents (license scans) stored in S3 with presigned URLs (15min expiry), never public URLs
- GPS history masked for completed trips after 30 days (only origin/destination retained for analytics)
- GDPR: data deletion job runs within 30 days of account deletion request
- Audit log for all PII access (immutable log to S3 + CloudTrail)

---

## 1️⃣2️⃣ Deployment Architecture

### Cloud Provider

**AWS** as primary. Multi-AZ within each region. Rationale: Mature managed services (RDS, ElastiCache, MSK/Kafka, EKS), global presence, best-in-class networking.

### Containerization & Orchestration

```yaml
# Each service packaged as Docker image
# Multi-stage builds → minimal production image (distroless or alpine)
# Base image pinned to specific SHA digest

# Kubernetes (EKS) cluster per region
# Namespace per environment (prod, staging, canary)

# Example deployment config:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: matching-engine
spec:
  replicas: 20
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 5
  template:
    spec:
      containers:
      - name: matching-engine
        image: uber-clone/matching:sha-abc123
        resources:
          requests: { cpu: "500m", memory: "512Mi" }
          limits:   { cpu: "2",    memory: "2Gi" }
        livenessProbe:
          httpGet: { path: /health, port: 8080 }
          initialDelaySeconds: 10
        readinessProbe:
          httpGet: { path: /ready, port: 8080 }
```

**HPA configs:** Matching engine scales on Kafka consumer lag metric. Location service scales on RPS metric.

### CI/CD Pipeline

```
Developer → GitHub PR
  → GitHub Actions:
    1. Unit tests + integration tests
    2. Static analysis (SonarQube)
    3. Security scan (Snyk, Trivy for container)
    4. Build + push Docker image (tagged with git SHA)
    
PR Merged → main:
    5. Deploy to staging (auto)
    6. Integration test suite
    7. Performance regression test (k6)
    
Release tag:
    8. Canary deploy (5% traffic) to prod
    9. Monitor error rate + latency for 30min
    10. Progressive rollout: 5% → 25% → 50% → 100%
    11. Automated rollback if error rate > 0.1% or p99 > SLO
```

### Observability Stack

```
Metrics:    Prometheus + Grafana (service metrics, business metrics)
Logs:       Structured JSON logs → Fluent Bit → Elasticsearch → Kibana
Tracing:    OpenTelemetry SDK → Jaeger (distributed tracing, 1% sampling)
Alerting:   PagerDuty integration (Prometheus AlertManager)
Profiling:  Continuous profiling (Pyroscope / AWS CodeGuru)
```

---

## 1️⃣3️⃣ Monitoring & Alerting

### Key Metrics (SLIs)

**Availability:**

- Successful ride completion rate (target: >99.5%)
- API success rate per service (target: >99.9%)

**Latency:**

- P50/P95/P99 of ride match time (target: P99 < 2s)
- P99 of location update ingestion (target: < 100ms)
- P99 of fare calculation (target: < 300ms)

**Business metrics (real-time dashboards):**

- Active rides right now
- Driver utilization rate (rides/available drivers)
- Surge zones on map
- Payment success rate
- Cancellation rate by reason

**Infrastructure:**

- Kafka consumer lag per topic/consumer group
- Redis memory utilization + eviction rate
- PostgreSQL replication lag
- Pod restart rate

### Alerting Strategy

```
P1 (page immediately, < 5min response):
  - Payment success rate < 95% over 5min window
  - Match success rate < 90%
  - API error rate > 1% for core services
  - Any region becomes unreachable

P2 (page on-call, < 30min response):
  - P99 latency > 2x SLO for 10 min
  - Kafka consumer lag > 100K messages
  - DB replication lag > 30s

P3 (Slack alert, next business day):
  - Driver utilization drops > 20% vs 7-day avg
  - Unusual cancellation rate spike
```

### SLOs

|Service|Availability SLO|Latency SLO|
|---|---|---|
|Core Ride Flow|99.99%|P99 match < 2s|
|Payment|99.999%|P99 < 3s|
|Location Updates|99.95%|P99 < 100ms|
|Notifications|99.9%|P99 < 2s|
|Trip History|99.9%|P99 < 500ms|

Error budget tracked monthly. If 50% of error budget consumed in <2 weeks, freeze non-critical deployments.

---

## 1️⃣4️⃣ Trade-offs & Alternatives

### Key Design Decisions

**H3 hexagonal grid for surge vs simple lat/lng bounding box:** H3 provides consistent-area cells enabling fair comparison across regions. Bounding boxes have irregular area at different latitudes. H3 also enables efficient neighbor cell lookup for surge propagation. Trade-off: H3 adds library dependency and conceptual complexity. Worth it for accuracy.

**Redis GEO for driver locations vs PostGIS:** Redis GEORADIUS is O(N+log(M)) and operates in memory — perfect for 125K updates/sec with sub-millisecond reads. PostGIS is durable and handles complex spatial queries but would require a massive PostgreSQL cluster to handle this write volume. We use PostGIS only for geofencing rules (city boundaries, airport zones) which are infrequent reads.

**Kafka vs direct RPC for trip events:** Kafka decouples producers from consumers — notification service can be down without affecting core trip flow. It enables replay for analytics backfill. The trade-off is operational complexity and eventual consistency. For a system this scale, the resilience benefit outweighs complexity.

**Why not a single PostgreSQL for everything:** Cassandra for location history is chosen because: (a) write throughput requirements (125K/sec) would overwhelm a single PG cluster; (b) location history is accessed by time range for a single driver — Cassandra's partition model is perfect; (c) we have no need for complex joins on location data.

**Why not MongoDB:** MongoDB's flexible schema is not needed — our data models are well-defined. PostgreSQL provides better ACID guarantees, mature replication, and a richer ecosystem for our relational data. Cassandra is better for our time-series use case than MongoDB.

**WebSocket vs SSE for driver location:** WebSocket chosen because we need bidirectional communication (future: in-app chat, driver status updates). SSE is simpler but unidirectional. Overhead difference is minimal.

### Future Improvements

- **ML-based dispatch:** Replace heuristic matching with a trained model predicting optimal driver assignments based on historical traffic patterns.
- **Carpooling / UberPool:** Requires multi-rider trip coordination, route optimization, fare splitting — significant complexity addition.
- **Driver earnings optimization:** Real-time guidance to drivers on where to position for maximum earnings.
- **Predictive surge:** Anticipate surge before it happens using event data (concerts, airports), pre-position drivers.
- **CRDT-based distributed state:** For eventual consistency of driver availability without Redis as single source of truth.
- **Edge computing:** Deploy location ingest endpoints at CDN edge nodes to reduce GPS ping latency globally.

---

## 1️⃣5️⃣ Step-by-Step Implementation Plan

### Phase 1 — MVP (Months 1–4): One City, Core Flow

**Goal:** Functional end-to-end ride booking in a single city with 1,000 concurrent users.

**Team:** 3 backend engineers, 2 mobile (iOS/Android), 1 DevOps, 1 PM

**Deliverables:**

- Monolith with modular internal structure (auth, trips, location, payments as modules, not separate services)
- PostgreSQL for all data, Redis for location geo-index
- Basic matching (nearest driver, no surge)
- Stripe integration for payments
- FCM push notifications
- Simple Kubernetes deployment (single region, single AZ)
- Basic Grafana dashboard

**Shortcuts taken (intentionally):**

- No surge pricing
- No driver payout batching (manual transfers)
- No trip history archival
- No rate limiting sophistication

---

### Phase 2 — Scale & Harden (Months 5–8): Multi-City, 10K Concurrent

**Goal:** Extract high-traffic services, add resilience, expand to 5 cities.

**Team:** Add 4 backend engineers, 1 data engineer, 1 SRE

**Deliverables:**

- Extract Location Service (separate deployment, gRPC)
- Extract Matching Engine
- Kafka cluster for event streaming
- Surge pricing (H3-based)
- Multi-AZ deployment
- Read replicas for PostgreSQL
- Cassandra for location history
- Circuit breakers + retry logic
- On-call rotation + PagerDuty
- Load testing framework (k6 scripts)

---

### Phase 3 — Reliability & Performance (Months 9–14): 100K Concurrent

**Goal:** Production-grade reliability, multi-region, 99.99% SLA.

**Team:** Add platform team (4 engineers), security engineer, data platform team

**Deliverables:**

- Multi-region deployment (2 regions)
- Extract all remaining microservices
- Implement full distributed tracing (OpenTelemetry + Jaeger)
- Advanced fraud detection integration
- GDPR compliance tools (data deletion, export)
- Chaos engineering (Chaos Monkey) — build confidence in fault tolerance
- Database sharding (Citus or Vitess)
- WebSocket gateway with Redis PubSub for real-time location streaming
- Automated canary deployments
- SLO dashboards + error budget tracking

---

### Phase 4 — Global Scale (Months 15–24): 500K+ Concurrent

**Goal:** Global expansion, ML-enhanced systems, self-healing infrastructure.

**Team:** Full platform org (20+ engineers across multiple teams)

**Deliverables:**

- All 5 regions active (active-active)
- ML-based matching model (replace heuristic ranker)
- Predictive surge pricing
- Driver positioning recommendations
- Global rate limiting (cross-region state)
- Data warehouse (Redshift/BigQuery) + real-time analytics (Apache Flink)
- Automated disaster recovery drills
- Service mesh upgrade (Istio → mature configuration)
- Cost optimization: reserved instances, spot instances for stateless pods

---

### Team Breakdown at Maturity

|Team|Ownership|
|---|---|
|Rider Platform|Rider app, rider-facing APIs|
|Driver Platform|Driver app, driver-facing APIs|
|Matching & Dispatch|Matching engine, surge pricing|
|Location Platform|Location service, geo-indexing|
|Payments|Payment service, payouts, fraud|
|Notifications|Push, SMS, email infrastructure|
|Data Platform|Kafka, Cassandra, data warehouse|
|Infrastructure / SRE|Kubernetes, CI/CD, observability|
|Security|Auth, compliance, pen testing|

---

## Summary

|Layer|Technology Choice|
|---|---|
|API Gateway|Kong + AWS ALB|
|Services|Go (high throughput) + Java (payments)|
|Service Mesh|Istio (mTLS, traffic management)|
|Realtime|WebSocket + Redis PubSub|
|Event Bus|Apache Kafka|
|Current State Cache|Redis Cluster (Geo + pubsub)|
|OLTP Database|PostgreSQL (Citus for sharding)|
|Time-Series / History|Apache Cassandra|
|Search|Elasticsearch|
|Object Storage|AWS S3|
|Container Orchestration|Kubernetes (EKS)|
|CI/CD|GitHub Actions + ArgoCD|
|Observability|Prometheus + Grafana + Jaeger|
|CDN|CloudFront|
|Payments|Stripe (tokenization)|
|Notifications|FCM/APNs + Twilio|

This blueprint gives you both the interview-ready conceptual clarity and the production-grade engineering depth to actually build it. The key insight holding it together: **different components have radically different scaling profiles**, and the architecture respects that by choosing the right data store, communication pattern, and consistency model for each domain independently.