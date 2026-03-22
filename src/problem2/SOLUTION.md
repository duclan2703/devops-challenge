# Castle In The Cloud — Highly Available Trading System on AWS

Design of a resilient, scalable, cost-effective trading platform inspired by core Binance-style features, with emphasis on availability, **correctness**, and scalability.
---

## 0. Constraints (as specified)

| Constraint | Value |
|------------|--------|
| **Cloud provider** | **Amazon Web Services (AWS)** only |
| **Throughput** | **500 requests per second** (aggregate API traffic) |
| **Latency** | **p99 response time < 100 ms** (API path, region-local clients) |

Non-functional paths (e.g. async analytics, batch exports) are excluded from the 100 ms p99 unless stated. **Order submission / matching** has stricter **business** SLOs documented in §8.

---

## 1. Features selected (scope vs breadth)

You cannot cover all of Binance; pick features that prove **HA, scalability, real-time behavior, and financial correctness**.

| Feature area | In this design | Why it matters |
|--------------|----------------|----------------|
| **Identity & API security** | **Cognito** JWT, **IAM** task roles, **KMS**, **WAF** | Trust boundary for money-moving APIs |
| **Spot trading / matching** | Single-writer **matching engine**, **Aurora** durability, **§3** | Core latency + **at-most-once** fills |
| **Order book & market data** | **MSK** + **WebSocket** fan-out, **sequence** IDs **§4.3** | Real-time + gap detection |
| **Wallet / balances** | **Reservations**, **Aurora** ACID **§4.4** | Prevents oversell |
| **Trade history & audit** | **order_events**, **S3** export **§4.2** | Compliance + replay |
| **Observability** | **X-Ray**, **CloudWatch**, business SLOs **§8** | p99 and fill-latency investigations |

**Out of scope (explicit):** full **KYC** pipelines, **fiat** rails, **margin/derivatives**, full **custody**—same boundary as a focused architecture exercise.

---

## 2. Architecture overview — diagrams and planes

The deliverable asks for an **overview of services and roles**. Flow: **§2.1** overview diagram → **§2.2** control/data plane → **§2.3** Multi-AZ → **§2.4** read/write → **§2.5** order-flow **sequence** → **§2.6** service index (details in **§5**).

### 2.1 Architecture overview diagram

Single **regional** view: how **users**, **edge**, **identity & secrets**, **application**, **data**, and **observability** connect. (VPC boundaries, per-AZ placement, and security dashed lines are expanded in §2.3.)

```mermaid
flowchart TB
    subgraph Client["Clients"]
        U[Web / mobile / API clients]
    end

    subgraph Edge["Edge & load balancing"]
        R53[Route 53]
        CF[CloudFront]
        WAF[WAF + Shield]
        ALB[ALB + ACM TLS]
    end

    subgraph Identity["Identity, secrets & encryption"]
        COG[Cognito OIDC/JWT]
        SM[Secrets Manager]
        KMS[KMS CMKs]
    end

    subgraph App["Application — private subnets, multi-AZ"]
        ECS[ECS Fargate API + WebSocket]
        ME[Matching engine + coordination]
    end

    subgraph Data["Data & messaging"]
        AUR[(Aurora PostgreSQL)]
        RED[(ElastiCache Redis)]
        MSK[[MSK Kafka]]
        S3[(S3 audit archive)]
    end

    subgraph Obs["Observability"]
        CW[CloudWatch]
        XY[X-Ray / OTel]
    end

    U --> R53 --> CF --> WAF --> ALB --> ECS --> ME
    U -.->|sign-in token| COG
    ECS --> AUR
    ECS --> RED
    ME --> AUR
    ME --> RED
    ME --> MSK
    ECS --> MSK
    ECS -.->|audit / compliance export| S3
    ECS -.-> SM
    ME -.-> SM
    AUR -.-> KMS
    S3 -.-> KMS
    ECS -.-> KMS
    ECS --> CW
    ECS --> XY
```

**How to read this diagram**

| Layer | Responsibility |
|-------|------------------|
| **Edge** | DNS, **TLS** termination (**ACM**), **WAF** / **Shield**, traffic distribution to **compute**. |
| **Identity** | **Cognito** issues **JWTs**; **Secrets Manager** holds **DB/MSK** credentials for **IAM**-scoped tasks; **KMS** keys **encrypt** Aurora, S3, and sensitive fields. |
| **Application** | **Stateless** **ECS** tier + **stateful matching** tier (see **§3** for consistency). |
| **Data** | **System of record** (**Aurora**), **cache** (**Redis**), **event bus** (**MSK**), **long-term audit** (**S3**). |
| **Observability** | **Metrics/alarms** and **distributed traces** — full order path in **§8**. |

### 2.2 Control plane vs data plane

| Plane | Responsibility | Typical AWS components |
|-------|----------------|-------------------------|
| **Control plane** | **Configuration and lifecycle**: deploys, autoscaling policies, secrets rotation, feature flags, **matching-engine leader election config** (not the hot path per order). | **ECS/EKS control plane**, **Systems Manager Parameter Store**, **Secrets Manager**, **CodePipeline/CodeDeploy**, optional **etcd** (if self-managed consensus). |
| **Data plane** | **Per-request / per-event** work: HTTP/WebSocket, **matching**, **persistence**, **market-data fan-out**. Everything that touches customer money or orders in real time. | **ALB**, **ECS tasks**, **matching-engine processes**, **Aurora**, **ElastiCache**, **MSK**, **X-Ray** |

**Rule:** Trading correctness and latency SLOs apply to the **data plane**. Control-plane outages may block deploys but must not silently corrupt order state.

### 2.3 Diagram A — Multi-AZ data plane (VPC, synchronous vs asynchronous hints)

```mermaid
flowchart LR
    subgraph Internet
        Clients[Clients]
    end

    subgraph Identity["Identity & security"]
        COG[Cognito User Pool OIDC/JWT]
        KMS[KMS CMKs]
        SM[Secrets Manager]
    end

    subgraph Edge
        R53[Route 53]
        CF[CloudFront]
        WAF[WAF]
        ALB["ALB + ACM TLS"]
    end

    subgraph AZ_A["AZ-a"]
        ECS_A[ECS API / WS IAM task role]
        ME_L[Matching engine LEADER]
        REDIS_A[ElastiCache replica TLS]
    end

    subgraph AZ_B["AZ-b"]
        ECS_B[ECS API / WS IAM task role]
        ME_S[Matching engine STANDBY]
        REDIS_B[ElastiCache replica TLS]
    end

    subgraph DataRegion["Regional / multi-AZ data stores"]
        AUR[(Aurora SSE-KMS)]
        MSK[[MSK TLS]]
        S3[(S3 audit SSE-KMS)]
    end

    Clients -.->|sign-in / refresh OIDC| COG
    Clients --> R53 --> CF --> WAF --> ALB
    ALB --> ECS_A
    ALB --> ECS_B
    ECS_A --> ME_L
    ECS_B --> ME_S
    ME_L --> AUR
    ME_L --> MSK
    ME_L -.->|hot cache snapshot| REDIS_A
    ME_S -.->|follower / warm| REDIS_B
    MSK -.->|async: market-data consumers| ECS_A
    ECS_A -.->|DB creds MSK creds| SM
    ME_L -.->|broker DB secrets| SM
    AUR -.->|encryption keys| KMS
    S3 -.->|encryption keys| KMS
    ECS_A -.->|decrypt data keys| KMS
```

**Legend (not all edges are HTTP):**

- **Solid user path:** Client → edge → **ECS** → **matching leader** → **Aurora** (durable commit) and **MSK** (async publish — see §2.4). **Private APIs** carry **Bearer JWT** issued by **Cognito**; **ECS** validates via **JWKS** (§5).
- **Dashed — security / support:** **Cognito** = login/token **out of band** of the main request line; **Secrets Manager** = runtime creds for tasks; **KMS** = **SSE-KMS** for Aurora/S3 and **data-key** use; **dashed** cache/async unchanged below.
- **Dashed — data:** Cache replicas; async fan-out. **ElastiCache** holds **working set / snapshots**, not the **source of truth** for recovery (§3).

### 2.4 Diagram B — Write path vs read path

| Path | Purpose | Synchronous steps (typical) | Async (does not block HTTP ack for ack-eligible APIs) |
|------|-----------|------------------------------|--------------------------------------------------------|
| **Write** | Submit / cancel order, **match** | ALB → ECS → **JWT validate (Cognito JWKS) + scopes** → **idempotency check (Redis)** → **matching engine** → **append audit + order state to Aurora** → **publish match event to MSK** | Downstream: WebSocket consumers, analytics, **MSK** replication |
| **Read** | Balances, public book, history | ALB → ECS → **JWT** for **private** routes → **Redis** (ticker, cached book) or **Aurora replica** (history) | — |

```mermaid
flowchart TB
    subgraph Write["WRITE PATH — latency-sensitive; must be correct"]
        W0["HTTPS + Bearer JWT"] --> Wauth["Validate JWT Cognito JWKS + scopes"]
        Wauth --> W1[POST /orders]
        W1 --> W2[Idempotency Redis]
        W2 --> W3[Matching engine leader]
        W3 --> W4["Aurora: WAL-equivalent order + match records"]
        W3 --> W5[MSK produce: MatchEvent with seq]
        W5 -.->|async| W6[WS + consumers]
    end

    subgraph Read["READ PATH — hot reads (p99 SLO)"]
        R0["GET + Bearer if private routes"] --> Rauth[Validate JWT when required]
        Rauth --> R1[GET /book /ticker]
        Rauth --> R3[GET /history /balances]
        R1 --> R2[Redis snapshot]
        R3 --> R4[Aurora replica]
    end
```

**MSK publish:** For **HTTP 200** semantics, the design **commits durable state in Aurora first**, then produces to MSK (or uses **outbox pattern** so MSK failure does not lose events — see §3). **p99 < 100 ms** assumes the **synchronous** segment stays within budget (§6).

### 2.5 Order flow — sequence diagram (critical path)

Simplified **happy path** (single region; **ECS** hosts API—**not** API Gateway WebSocket as in some EKS-centric designs). Matches **§2.4** write path.

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Edge ALB+WAF
    participant A as ECS API
    participant R as Redis
    participant M as Matching leader
    participant D as Aurora
    participant K as MSK

    C->>E: HTTPS POST /orders Bearer JWT
    E->>A: Forward
    A->>A: Validate JWT Cognito JWKS
    A->>R: Idempotency key check
    A->>M: Route order to shard
    M->>D: Txn: orders fills reservations outbox
    M->>R: Update hot book snapshot async
    D-->>M: Commit OK
    M->>K: Produce match event async
    M-->>A: Ack
    A-->>C: 200 order_id
    Note over K: Consumers fan-out WS market data
```

### 2.6 Service index (detail in §5)

| Tier | AWS components |
|------|----------------|
| **Edge** | Route 53, CloudFront, WAF, **ALB + ACM** |
| **Identity & secrets** | Cognito, IAM task roles, Secrets Manager, KMS |
| **Compute** | **ECS Fargate** (API/WS); **EC2/ECS** (matching + **etcd**) |
| **Data** | Aurora + RDS Proxy, ElastiCache, MSK, S3 |
| **Observability** | CloudWatch, X-Ray |

Full **rationale and alternatives** table is in **§5** (avoids duplicating edge/data rows here).

---

## 3. Matching engine — consistency, persistence, and failover

This is the **core** of the system; web-tier HA is insufficient without a precise **matching** model.

### 3.1 Consistency model and the “at-most-once fill” problem

**Requirement:** A match (fill) must be **at-most-once** from the client’s perspective: **never double-execute** the same fill; **never lose** an accepted order without explicit **rejection** or **durably recorded** state so the client can reconcile.

| Scenario | Desired behavior |
|----------|------------------|
| Leader **accepts** an order in memory, **crashes** before **durable commit** | Client may **retry with same idempotency key** (§4); engine **must not** double-match. **Recovery:** uncommitted in-memory state is **lost**; client sees **timeout** and retries — idempotency dedupes **submission**, **matching** replays from **durable intent** in Aurora. |
| Leader **commits** match to Aurora, **crashes** before MSK publish | **Outbox pattern:** row in `outbox` table in **same transaction** as match; **relay** publishes to MSK; no lost fills. |
| Network **partition** between AZs | See **split-brain** (§7.3). |

**Implementation sketch:**

1. **Order intent** is **inserted** into Aurora (`orders` with status `PENDING`) in a **transaction** **before** the in-memory book is updated (or in the same critical section with **two-phase** semantics).
2. **Matching** updates **same transaction**: `orders` → `FILLED`, `fills` insert, **balances / reservations** updated.
3. **Commit** to Aurora = **durable** decision; only then is the match **irreversible** for billing.

**Redis is not the durability boundary.** If Redis dies, **replay** from Aurora + **rebuild** in-memory book from **event log** or **snapshot + delta** (§3.2).

### 3.2 How the order book is persisted

| Layer | What it stores | Durability |
|-------|----------------|------------|
| **Aurora (primary)** | **Append-only** `order_events` (or equivalent), **current order state**, **fills**, **balance reservations**, **outbox rows** | **Durable**; Multi-AZ sync per Aurora config |
| **Optional: MSK as immutable log** | **Match events** for **external** consumers; **not** required for engine recovery if Aurora is source of truth | Replicated |
| **ElastiCache** | **Hot order book** snapshot, **ranked queues** per symbol for speed | **Ephemeral**; rebuilt after failover |

**Recovery path if Redis fails before DB write completes:**

- **Invariant:** No fill is **acknowledged to the client** until **Aurora commit** succeeds. If the process dies **after** commit but **before** Redis update, **new leader** loads state from Aurora and **rehydrates** Redis from **DB + optional event replay**.
- If the process dies **before** commit, the **order** may be `PENDING` or absent — client **idempotency retry** converges.

**WAL terminology:** Aurora’s **storage layer** provides **redo**; **application-level** “WAL” is the **append-only order_events + outbox** in PostgreSQL (or **Kinesis** as **ingest** in larger systems — here Aurora suffices at 500 RPS).

### 3.3 Leader, standby, and concrete consensus

**Options (AWS-grounded):**

| Approach | How it works | Operational reality |
|----------|--------------|---------------------|
| **Dedicated EC2 + embedded Raft** (e.g. **Hashicorp Raft** / **etcd** as separate cluster) | **3+ nodes** for quorum; **leader** runs matcher | **etcd is not a managed AWS service**; typical patterns: **self-managed etcd on EC2** (3-node ASG across AZs), or **EKS** (Kubernetes uses etcd for the control plane — you don’t run application Raft inside it unless you deploy **etcd as workloads**). For matching-only, **small EC2 cluster + etcd** is the clearest **ops** story. |
| **ECS + external lock** | **Single writer** via **DynamoDB conditional writes** / **RDS advisory locks** | Simpler but **lock TTL** and **fencing** must be bulletproof — often **weaker** than Raft for **hard** partition tolerance |
| **Single active + Aurora row lease** | **Leader lease** with **expiry** and **fencing token** in DB | Works at moderate scale; **must** fence stale leaders |

**Recommendation for this design:** **Matching engine on EC2 (or ECS with stable ENIs)** in **3-node** group across AZs, using **etcd** (self-managed on EC2 or **EKS** if you already run K8s) **or** **AWS-managed alternative**: run **only** the coordination on **small EC2 ASG** with **etcd**, **not** “Raft in a sentence.”

**Raft named =** typically **etcd** (Go Raft implementation) **or** Consul; **document which** and **how many** nodes, **which AZs**.

### 3.4 Failover RTO / RPO

| Metric | Target (example — tune per product) | Notes |
|--------|-------------------------------------|--------|
| **RPO** | **~0** for committed matches (Aurora sync) | Uncommitted in-flight may be **lost** from client POV → **idempotency** |
| **RTO** | **Seconds to low tens of seconds** | **Leader election** (etcd) + **standby** catch-up + **resume** ingestion; **not** instant — **document** e.g. **15–45 s** worst case so stakeholders know **trading may halt** that window |
| **During RTO** | **Reject or queue** new orders at API | **503** with `Retry-After` or **read-only** flag (§7.4) |

**30 seconds of halted matching** is a **P0** for a live exchange; mitigation = **symbol sharding** (only **affected** symbols down), **multi-region** (out of scope for minimal design), **transparent** status page + **runbook**.

---

## 4. Financial-domain correctness

Requirements that **generic** web stacks often omit.

### 4.1 Idempotency keys (order submission)

- Clients send **`Idempotency-Key`** (or body field) on **POST /orders**.
- **Redis**: `SET key UUID NX EX 86400` (or store hash of request) — **duplicate** returns **same** `order_id` / **409** with prior response body cached **if** required.
- **Prevents:** duplicate orders on **client retry after timeout**.

### 4.2 Immutable audit trail (separate concern from “app DB”)

- **Every** state transition: `PENDING` → `PARTIAL` → `FILLED` / `CANCELLED` appended to **`order_events`** (append-only table) **in the same transaction** as state change **before** HTTP response (or via **strict** ordering).
- **Regulatory / forensic:** periodic **export to S3** (immutable bucket, **Object Lock** optional), **separate** from **operational** queries on `orders` main table.
- **Not** “same as application DB” — **append-only** stream is the **legal** timeline; **current row** is a **projection**.

### 4.3 Sequence numbers and market-data integrity

- Each **symbol** has a monotonic **`sequence_id`** (or **Kafka offset + partition**) on **every** book update / trade published to **MSK**.
- **Consumers** (WebSocket fan-out, external feeds) **detect gaps** → **request snapshot** via **REST** `GET /book?symbol=X&after_seq=N` and **resync**.
- **MSK** partition key = **symbol** to preserve **per-symbol** ordering.

### 4.4 Balance locking and oversell prevention

- **Never** “read balance then write later” without **locking** or **reservation**.
- **Models:**
  - **Pessimistic:** `SELECT … FOR UPDATE` on **balance row** in Aurora in same txn as **reservation insert**.
  - **Reservation:** decrement **available**, increment **reserved** atomically; **match** moves **reserved** → **filled**.
- **Concurrent orders** from one account: **serialized** per **account_id** (shard lock) or **DB constraint** so **sum(reservations) ≤ balance**.

### 4.5 MSK topic design (Kafka)

**Partition key:** **`symbol`** (or **symbol shard**) so **per-symbol** order is preserved. Example topic names:

| Topic | Typical producers | Consumers |
|-------|-------------------|-----------|
| `orders.submitted` | API (after idempotency) | Matching engine |
| `orders.matched` | Matching engine | Wallet projection, history, **market data** |
| `orders.cancelled` | Matching / API | Wallet, notifications |
| `market.trades` | Matching | WebSocket / external feeds |
| `market.book.delta` | Matching | Real-time depth (optional vs snapshot API) |

**Note:** Durable **truth** remains **Aurora**; MSK is for **ordered fan-out** and **replay** (see **outbox** in §3 if producer must not lose events).

---

## 5. AWS services, rationale, and alternatives *within AWS*

**Clarification vs earlier sketches:** **ElastiCache** is **not** the durable order-book store — it holds **hot snapshots**, **idempotency keys**, and **reservations**; **Aurora** (+ append-only events / outbox) is the **source of truth** (§3.2).

### Edge and networking

| AWS service | Role | Why this service | Other AWS options considered |
|-------------|------|------------------|------------------------------|
| **Route 53** | Public DNS (ALIAS to CloudFront/ALB); optional private hosted zones | Latency-based / weighted / failover routing | Registrar DNS only (weaker AWS integration) |
| **CloudFront** | Static assets; optional cache for idempotent GETs | Edge TLS, offload | ALB-only |
| **AWS WAF + Shield** | L7 rules, rate limits, DDoS | Native on ALB/CloudFront | NACLs/SGs only (no L7) |
| **ALB** | TLS, HTTP/WebSocket routing, **sticky sessions** for WS (best-effort) | Health checks, cross-zone | NLB if raw TCP only |

### VPC and internal traffic

| AWS service | Role | Why | Other AWS options |
|-------------|------|-----|-------------------|
| **VPC, subnets** | Public (ALB, NAT); private (ECS, RDS, Redis, MSK) | Blast-radius | Default VPC (non-prod) |
| **IGW / NAT / VPC endpoints** | Inbound path; outbound; private AWS API access | Cost + security | NAT instance (legacy) |
| **Security groups** | Least-privilege east–west | Stateful | NACLs optional |

### Compute

| AWS service | Role | Why | Other AWS options |
|-------------|------|-----|-------------------|
| **ECS Fargate** | Stateless API + WebSocket | Ops simplicity, multi-AZ | **EKS** if you need many microservices + **LB controller**; **EC2 ASG** for bare-metal control |
| **EC2 (e.g. `c7i`) + ASG** | **Matching engine** + optional **etcd** nodes | Predictable CPU, placement groups for latency | Fargate for matching if variance acceptable |

### Data and messaging

| AWS service | Role | Why | Other AWS options |
|-------------|------|-----|-------------------|
| **Aurora PostgreSQL** | Orders, balances, **order_events**, **outbox** | Durable, failover, replicas | DynamoDB for specialized KV |
| **RDS Proxy** | Pooling, failover handling | Tail latency | Direct connect without proxy |
| **ElastiCache Redis** | Idempotency, reservations, **hot book** (non-durable) | Sub-ms | DAX if DynamoDB |
| **MSK (Kafka)** | Sequenced match/book events; consumer lag metrics | Ordering + replay | Kinesis, SQS (different semantics) |
| **S3 + Object Lock** | Long-term **audit** exports | Compliance, cheap | — |

### Observability and incident response

| AWS service | Role | Why | Other AWS options |
|-------------|------|-----|-------------------|
| **CloudWatch** | Metrics, **alarms → SNS** | Native | — |
| **X-Ray** or **ADOT** | **Distributed traces** (§8) | p99 root cause | Self-hosted Jaeger |
| **SNS** | Fan-out to **PagerDuty** / email / Lambda | Escalation | EventBridge for complex routing |
| **Managed Grafana** (optional) | Dashboards over **CloudWatch** / **X-Ray** | Single pane for ops + SLOs | Self-hosted Grafana on ECS/EKS |

### Security and identity (consolidated)

| Area | Approach | AWS |
|------|------------|-----|
| **Humans / CI** | SSO, no static keys in repos | **IAM Identity Center**, **OIDC** to CI **→** **IAM roles** |
| **Workloads** | **Task roles** / instance profiles; least privilege to **S3/KMS/MSK** only | **IAM**, **STS** |
| **End users** | **JWT** (**Cognito** User Pools); scopes (`orders:write`, …); **server-side** `sub` → **account** binding | **Cognito** |
| **Secrets** | DB/MSK creds **rotated** where possible | **Secrets Manager** + **RDS Proxy** integration |
| **Config flags** | `TRADING_HALTED`, etc. | **Parameter Store** **SecureString** + **KMS**; **IAM** who can change (**§7.4**) |
| **Encryption** | TLS everywhere (ALB/CF/Redis/MSK); **SSE-KMS** Aurora & S3; **EBS** volumes | **ACM**, **KMS** |
| **Audit / detect** | Infra changes + data-plane audit | **CloudTrail** org trail; Aurora audit logs; **order_events** **§4.2**; optional **GuardDuty**, **VPC Flow Logs** |
| **SDLC** | Supply-chain | **ECR** scan, **IaC** scan (**Checkov**), pen test policy |

**Network:** **SGs** (ALB→ECS→data only); **VPC endpoints** for AWS APIs; **WAF** rate/geo rules at edge (§5 Edge table).

---

## 6. Throughput, and end-to-end latency budget (p99 < 100 ms)

### 6.1 Throughput: 500 RPS

- **Headroom:** **2–4 Fargate tasks**, **Application Auto Scaling** on `RequestCountPerTarget` or CPU.
- **CloudFront** for cacheable GETs reduces origin RPS.

### 6.2 Latency budget — **validated** claim, not hope

**Scope:** **Read-heavy** synchronous API (e.g. **GET ticker**, **GET shallow book** from Redis) — the **stated** p99 < 100 ms target. **Order submit** that **touches Aurora sync write + match** may be **higher** unless optimized; **separate** business SLO for **ack-to-client** (§8).

| Segment | Budget (typical p50 / **allow for p99 tail**) | Notes |
|---------|-----------------------------------------------|--------|
| **ALB + TLS** | ~0.5–2 ms / **~1–3 ms** | Cross-AZ hop adds **~1 ms** if client not co-located |
| **ECS processing + JSON** | ~2–5 ms / **~5–12 ms** | GC, serialization — **right-size** tasks |
| **Redis round-trip** | ~0.2–1 ms / **~1–3 ms** | Same-AZ |
| **RDS Proxy + Aurora read** | ~1–3 ms / **~5–20 ms** | **Writes** higher: **~10–25 ms** sync commit Multi-AZ typical |
| **MSK** | **0 ms on critical path** if **async** after commit | **Fire-and-forget** from outbox relay — **must not** block HTTP if SLO is read path |

**Example — hot read path (GET, Redis hit):**  
~1 + ~6 + ~2 + ~0 = **~9 ms p50**, **~25–40 ms p99** with tail — **within 100 ms** if no Aurora on path.

**Example — write path (order submit, Aurora commit in request):**  
ALB + ECS + Redis idempotency + **Aurora txn** + **no blocking MSK** ≈ **25–50 ms p50**, **~60–90 ms p99** in a tuned region — **still feasible** for < 100 ms **if** matching is **fast** and **one** round-trip to DB; **leave ~10–40 ms** for **GC, spikes, cold pool**.

**Explicit:** **p99 < 100 ms** as a **platform** constraint should be **scoped** (e.g. **“public read APIs”** vs **“order accept ack”**) — both **budgeted** separately in production.

### 6.3 Validation

- **k6** with **client-side** p99, **X-Ray** waterfall (§8).
- **CloudWatch** ALB `TargetResponseTime` **per target group** — **not** sole source of truth.

---

## 7. High availability — engineered, not only “multi-AZ”

This section defines the **high availability (HA) design** for the trading data plane. The **primary** pattern is **Multi-AZ architecture** within a single **AWS Region**: isolate failure to an **Availability Zone** (AZ) without losing the **region**, while **matching-engine** correctness rules (§3, §7.3) still apply.

### 7.1 High availability design — Multi-AZ architecture

**Goals**

| Goal | What Multi-AZ delivers |
|------|-------------------------|
| **Fault isolation** | Loss of **one** AZ (power, networking, single-AZ service incident) should **not** take down the **entire** trading stack. |
| **Continued ingress** | **ALB** keeps routing to **healthy targets** in surviving AZs (**cross-zone load balancing** enabled). |
| **Data durability** | **Aurora** / **MSK** / **ElastiCache Multi-AZ** replicate or failover **within the region** per service model. |
| **Operational clarity** | Each tier’s **failure domain** is documented (below) so **on-call** knows what **auto-recovers** vs what needs **incident runbooks** or manual steps. |

**VPC layout (conceptual)**

- **One VPC** per **region** (e.g. `ap-southeast-1`), **non-overlapping** private CIDRs.
- **Minimum two AZs** in use: e.g. **`ap-southeast-1a`** and **`ap-southeast-1b`** (use **real AZ IDs** that map to **different** physical locations for your account).
- **Per AZ:** **public subnets** (ALB ENIs, NAT Gateway) + **private subnets** (ECS tasks, matching engine, **ElastiCache**, **MSK** broker ENIs, **Aurora** via managed placement).
- **Route tables:** Public subnets → **Internet Gateway**; private subnets → **NAT Gateway** (per AZ or shared per cost/HA trade-off) + **VPC endpoints** for AWS APIs.

*(Subnet topology is also shown at **data-plane** level in **§2.3**; no second duplicate diagram here.)*

**Multi-AZ placement by component**

| Layer | Multi-AZ pattern | If one AZ is impaired |
|-------|------------------|------------------------|
| **Route 53 / CloudFront / WAF** | **Regional** edge services; not AZ-scoped | Rare full-edge outages; **failover** to another region is a **separate** DR design (§9). |
| **ALB** | **Multi-AZ** by default; targets registered **per AZ** | **Unhealthy targets** in bad AZ **removed**; traffic flows to **other** AZs (**cross-zone** ON). |
| **ECS Fargate / EC2 workers** | **Tasks/instances** spread across **≥2 AZs** (capacity providers / ASG **AZ rebalancing**) | **Scheduler** replaces tasks; **min healthy %** on deploy avoids **total drain**. |
| **Matching engine + etcd** | **Leader + followers** across **AZs** (§3.3); **quorum** survives **one** AZ loss if **3+ nodes** | **Election** or **read-only** until quorum (**§7.3**); **RTO** in §3.4. |
| **Aurora PostgreSQL** | **Cluster volume** replicated **across AZs**; **primary** in one AZ, **failover** to replica | **Automatic failover** (typically **~30–120 s** for detection + promotion — **document** for SLOs). |
| **RDS Proxy** | **Multi-AZ** endpoint; **sticky** to **writer** / **reader** routing | **Reconnect** storm mitigation; **failover** handling without **app** hardcoding DB hostname churn. |
| **ElastiCache Redis** | **Multi-AZ with replica**; **automatic failover** of primary | **Brief** unavailability or **read-only** replica promotion; **cache rebuilt** from **Aurora** if needed (§3.2). |
| **MSK (Kafka)** | **Brokers** across **≥3 AZs** (recommended); **replication factor** ≥ 3 | **Partition** **under-replicated** until broker back; **producers** retry (**outbox** preserves **durability** — §3). |

**ALB cross-zone load balancing**

- **Enable** cross-zone on target groups so **clients** are not stuck if their **AZ** has **fewer** healthy targets.
- **Trade-off:** Slightly **higher** cross-AZ **latency** (~1 ms) vs **AZ-local-only** routing — for **500 RPS** and **p99** SLO, cross-zone is usually **acceptable** (§6).

**What Multi-AZ does *not* solve**

| Limitation | Implication |
|------------|-------------|
| **Region-wide** failure | **Multi-AZ** ≠ **multi-region**; **DR** second region + **Route 53** failover / **pilot light** stack (**§9**). |
| **Logical bugs** | **Duplicate fills**, **bad balances** — **correctness** in **§3–4**, not fixed by AZ redundancy. |
| **Split-brain matching** | **Network partition** between AZs — **§7.3**; **fail closed** over **double trading**. |

**Summary checklist (design review)**

- [ ] **≥2 AZs** used for **compute** and **stateful** data (**Aurora**, **MSK**, **ElastiCache** configs reviewed).  
- [ ] **ALB** health checks + **cross-zone** + **connection draining** on deploy.  
- [ ] **Matching** **quorum** layout documented (**3+** nodes across AZs).  
- [ ] **Runbook**: **Aurora failover** + **matching** **no-leader** behavior + **customer messaging** (status page).

### 7.2 WebSocket affinity and failover

| Issue | Mitigation |
|-------|------------|
| **ECS task dies** | **Sticky sessions** help only that task — on death, **connections drop**. |
| **Client behavior** | **Exponential backoff reconnect**; **resume** from **`Last-Event-ID` / sequence** (§4.3). |
| **Missed events during reconnect** | **Gap detection** on sequence → **snapshot resync** REST call. |
| **Grace period** | Document **max silence** before client must **snapshot**; server may send **heartbeat** + **seq**. |

**ALB sticky** is **best-effort**; **stateless WS** design: **subscribe** after reconnect with **from_seq**.

### 7.3 Split-brain and matching engine partition

| Scenario | Policy |
|----------|--------|
| **Leader** loses **quorum** (AZ link flap) | **Step down** — **no** match without **lease** + **fencing**; **reject** orders with **503** rather than **risk double match**. |
| **Standby** thinks it’s leader | **etcd/Raft** prevents **two leaders** with **votes**; **old leader** must **stop** writing (**fenced** via **monotonic token** in Aurora). |
| **Orders during uncertainty** | **Fail closed**: **better** short **unavailability** than **duplicate fills**. |

### 7.4 Read-only mode — concrete mechanics

**Trigger (examples — tune in runbooks):**

| Signal | Threshold (example) | Action |
|--------|----------------------|--------|
| Aurora **primary** unhealthy / **failover** in progress | **CloudWatch** RDS `DatabaseConnections` + **event** `RDS-EVENT` | **API middleware** flips **`TRADING_HALTED=true`** in **Parameter Store** / **feature flag**; **POST /orders** → **503** with JSON body `{ "code": "READ_ONLY" }` |
| Matching engine **no leader** > **N seconds** | **Health check** + **etcd** metrics | Same |
| **Error rate** from matching **> 5%** for **2 min** | **CloudWatch alarm** | **Circuit**: stop **new** orders; **GET** still allowed if **replicas** healthy |

**Implementation:** **Envoy/NGINX** or **app** checks **flag**; **no** vague “circuit breakers” without **numbers**.

---

## 8. Observability — beyond table stakes

### 8.1 Distributed tracing (required for p99 investigations)

- **AWS X-Ray** (or **OpenTelemetry** → **ADOT** → X-Ray/Jaeger) on **full write path**: **ALB segment** → **ECS** → **Redis** → **Aurora** → **outbox relay** → **MSK**.
- **Sampling:** **100%** for **errors**; **1–10%** for success at scale — **always** trace **slow** requests (**tail sampling**).

Without a **single waterfall**, p99 spikes are **guesswork**.

### 8.2 Business-level SLOs and alerts

| SLO (examples) | Measure | Not a default CloudWatch metric |
|----------------|---------|--------------------------------|
| **Order accept latency** | Time from **ingress** to **HTTP 200** with `order_id` | Custom **app timer** + X-Ray |
| **Match-to-WS lag** | `match_commit_time` → **first WS frame** with same `seq` | **MSK consumer lag** + **app** |
| **WebSocket drop rate** | Connections closed / min | **ALB** `ClientTLSNegotiationError` + **app** |
| **Fill correctness** | **Reconciliation** job: **sum(fills) = order_events** | **Batch** + **alarm** on mismatch **> 0** |

### 8.3 Alerting pipeline and escalation

| Component | AWS pattern |
|-----------|-------------|
| **Alarms** | **CloudWatch Alarm** → **SNS topic** `critical-trading` |
| **Paging** | **SNS** → **PagerDuty** / **Opsgenie** integration (or **Lambda** → webhook) |
| **Runbooks** | Link in **alarm description**; **on-call rotation** in PagerDuty |
| **Dashboards** | **CloudWatch** + **X-Ray** service map + **business SLO** panel; optional **Amazon Managed Grafana** (§5) for unified ops views |

**Technical-only** “ALB p99” is **insufficient** for **trading** — **composite** pages for **SLO burn rate**.

---

## 9. Scaling when the product grows

| Phase | Rough scale | Actions |
|-------|-------------|---------|
| **1 — Current** | **~500 req/s** | **ECS** task count + **ALB**; **Aurora** rightsized; **MSK** partitions **≈ symbols**; **Redis** cluster mode if needed |
| **2 — 10×** | **~5k req/s** | **HPA**/target-tracking on ECS; **Aurora** read replicas + **RDS Proxy** pool tuning; **MSK** add brokers/partitions; **symbol-sharded** matching **routing** |
| **3 — 100×+** | **50k+ req/s** | **Dedicated** matching on **EC2** (or **EKS** dedicated pools); **DB sharding** / **DynamoDB** for hot paths *only if modeled*; **NLB** for specific paths; **multi-region** active/active *only* with clear **book** boundaries |
| **Global** | DR / latency | **Route 53** latency/failover; **pilot-light** or **warm** secondary region; **per-region** order books (no silent **global** consistency) |

### 9.1 Monthly cost estimate (~500 req/s, single region)

**Disclaimer:** Figures are **order-of-magnitude** using **US East (N. Virginia)**–style **on-demand** list pricing; **your** bill varies by region, **data transfer**, **storage growth**, **MSK** sizing, and **discounts**. Always model in the **[AWS Pricing Calculator](https://calculator.aws/)**.

**Assumptions for this estimate**

| Assumption | Value used |
|------------|------------|
| Hours / month | **730** |
| Region | **us-east-1** (cheaper than many APAC/EU; use calculator for yours) |
| Discounts | **None** (no Savings Plans / RIs / Enterprise Discount) |
| Architecture | **ALB + ECS Fargate** (API), **3× EC2** (matching + etcd, **c7i.large**), **Aurora** cluster **db.r6g.large** (1 writer), **RDS Proxy**, **ElastiCache** **2× cache.r6g.large**, **MSK** **3× kafka.m5.large**, **2× NAT Gateway**, **CloudFront + WAF + Route 53**, moderate logs/traces |

**Rough monthly breakdown (USD)**

| Line item | How estimated | ~ USD/mo |
|-----------|----------------|----------|
| **NAT Gateway** | 2 × (~$0.045/h × 730h) + ~$20 **processing** | **~85** |
| **ALB** | ~$0.0225/h × 730 + ~$25 **LCU** (light) | **~42** |
| **ECS Fargate** (API) | 4 tasks × 0.5 vCPU, 1 GiB: ~(0.5×$0.0405 + 1×$0.0044)×730×4 | **~73** |
| **EC2** (matching/etcd) | 3 × **c7i.large** ~$0.0896/h × 730 (on-demand) | **~196** |
| **Aurora PostgreSQL** | **db.r6g.large** ~$0.29/h × 730 + ~100 GB storage × ~$0.10/GB | **~220** |
| **RDS Proxy** | Aurora Proxy ~$0.015/vCPU-h × **2** vCPU × 730 (varies) | **~22** |
| **ElastiCache** | 2 × **cache.r6g.large** ~$0.126/h × 730 | **~184** |
| **MSK** | 3 × **kafka.m5.large** ~$0.21/h × 730 + **broker storage** (~$0.10/GB-mo × ~300 GB) | **~490** |
| **CloudFront + WAF + Route 53** | Light traffic + 1 hosted zone + a few WAF rules | **~90** |
| **CloudWatch + X-Ray** | Metrics, moderate log ingest, sampled traces | **~80** |
| **Secrets Manager + KMS + Cognito** | Few secrets, keys, modest MAU | **~35** |
| **S3 + data transfer** | Audit + logs egress buffer | **~40** |

**Indicative total:** **~$1,550 / month** → rounds to **“low thousands” (~$1.5k–$2k)** for this **HA-ish** footprint.

**What moves the number most**

| Driver | Effect |
|--------|--------|
| **MSK** (broker class × count × storage) | Often **~25–35%** of total; **smaller brokers** or **MSK Serverless** (where fit) **cuts** hundreds. |
| **NAT** | **Second NAT** is almost **double** fixed NAT cost; **VPC endpoints** for S3/ECR reduce **NAT** data charges. |
| **Aurora** | **db.r6g.large** is not tiny; **db.t4g** / **Serverless v2** with low ACU **drops** hundreds (with **trade-offs**). |
| **Matching on Fargate vs EC2** | Swapping **3× c7i.large** for **Fargate** tasks changes **~$150–250** depending on size. |
| **Savings Plans / RIs** | Often **~30–40%** off **Fargate/EC2**; **Reserved** for Aurora/ElastiCache if **steady**. |

**Lean dev/staging** (smaller Aurora, **1 NAT**, **smaller MSK** or fewer brokers, no HA Redis): often **~$700–$1,000/mo** before discounts.

**Bottom line:** **“Low thousands USD/month”** here means roughly **~$1.0k–$2.2k/mo** for a **production-shaped** single-region stack with **MSK + Multi-AZ-ish** data tier; **not** a guarantee—**calculate** before committing.

---

## 10. How this design maps to the constraints

| Constraint | How the design satisfies it |
|------------|-----------------------------|
| **AWS only** | **Route 53, CloudFront, WAF, VPC, ALB, ECS, EC2 (matching/etcd), Aurora, RDS Proxy, ElastiCache, MSK, S3, X-Ray, CloudWatch, SNS, Parameter Store** — data plane as in §2–5; **security/identity** as in **§5 (Security and identity)**: **IAM / IAM Identity Center**, **Cognito**, **Secrets Manager**, **KMS**, **ACM**, **CloudTrail**. |
| **500 RPS** | §6.1 — small ECS footprint with headroom. |
| **p99 < 100 ms** | §6.2 **end-to-end budget**; scoped to **read** vs **write** paths; validated with **k6 + X-Ray** (§8). |
| **High availability** | §7.1 **Multi-AZ architecture** (VPC, ALB, ECS, Aurora, MSK, ElastiCache, matching quorum); §7.2–7.4 **session**, **partition**, **degraded** modes. |

**Cost:** Indicative **~$1.5k–$2k/mo** on-demand for a **HA-shaped** stack at **500 req/s** (see **§9.1**); **Savings Plans** / **rightsizing** reduce this.

---

## References in this document

| § | Topic |
|---|--------|
| §2 | Overview (§2.1), planes (§2.2), Multi-AZ (§2.3), read/write (§2.4), **sequence** (§2.5), service index (§2.6); full tables **§5** |
| §3 | Matching engine: consistency, persistence, Raft/etcd, RTO |
| §4 | Idempotency, audit, sequencing, balances |
| §5 | AWS services **+ Security and identity** (IAM, Cognito, encryption, audit) |
| §6 | Latency budget |
| §7 | **HA design: Multi-AZ** (§7.1), WebSocket, split-brain, read-only mode |
| §8 | Observability: X-Ray, business SLOs, PagerDuty/SNS |
| §9 | Scaling, multi-region, **monthly cost estimate (§9.1)** |
| §10 | Constraint mapping |