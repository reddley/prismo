# Backlog

## DONE:

**P0 — Critical Architecture**

~~### Migrate to PostgreSQL‑only storage~~
~~* **Why:** The current in‑memory implementation is incomplete and loses data on restart. Persisting all data in Postgres eliminates missing methods and ensures persistence.~~
~~* **What to do:** Finalise the database schema, implement a repository layer, and remove all `MemStorage` code paths. Create indexes on frequently queried fields.~~
~~* **Done when:** Every read and write goes through Postgres; data survives a cold start; indexes exist on hot query columns.~~

~~### Fix TypeScript schema/interface errors~~
~~* **Why:** Unresolved type errors can hide mismatched assumptions and cause runtime failures.~~
~~* **What to do:** Align TypeScript types with the actual database schema, implement missing interface methods, and enforce strict type checking in continuous integration.~~
~~* **Done when:** Running `tsc --noEmit` reports no diagnostics and the CI pipeline blocks new type errors.~~

~~### Connection pooling~~
~~* **Why:** Opening and closing database connections for every request becomes expensive as traffic grows. A connection pooler (e.g. PgBouncer) maintains a small number of open connections and reuses them, reducing overhead.~~
~~* **What to do:** Deploy a lightweight connection pooler in front of PostgreSQL, configure sensible limits for maximum and idle connections, and ensure every service uses the pool.~~
~~* **Done when:** Connections remain stable under load and p95 latency on hot queries stays below 300 ms.~~

~~### Job queue with idempotency~~
~~* **Why:** Ingestion and AI enrichment can produce duplicate work if retries occur. Without idempotency keys, the same task might be processed twice, causing duplicate writes or charges.~~
~~* **What to do:** Use a Redis‑backed queue (e.g. BullMQ). Include an idempotency key in each job payload. Implement exponential back‑off for retries and a dead‑letter queue for jobs that continually fail.~~
~~* **Done when:** Replaying a job does not create duplicate records; retries are limited; failed jobs are visible in a dead‑letter queue.~~

~~### Auth hardening~~
~~* **Why:** A surge of signups can lead to abuse or compromised accounts. Strong authentication mitigates risk.~~
~~* **What to do:** Use email verification queued through the job system. Apply per‑IP and per‑email signup throttling. Require CAPTCHA/Turnstile for registration. Hash passwords with Argon2id.~~
~~* **Done when:** Verification emails arrive within a minute; brute‑force attempts are blocked; sessions are secure and short‑lived.~~

~~### Rate limits & backpressure~~
~~* **Why:** A single tenant or user could overload the system. Without limits, critical endpoints may degrade for everyone.~~
~~* **What to do:** Implement a token‑bucket rate limiter at the API layer that applies per‑tenant quotas. Shed non‑critical work first (e.g. enrichment) and return HTTP 429 with `Retry‑After` headers.~~
~~* **Done when:** Under simulated load the system refuses excess traffic gracefully while maintaining quality of service for compliant tenants.~~

~~### Observability baseline~~
~~* **Why:** Without logs, metrics and traces it’s impossible to diagnose slowness or failures.~~
~~* **What to do:** Emit structured JSON logs, instrument RED/USE metrics, and propagate distributed traces via OpenTelemetry. Define service level objectives and configure alerts.~~
~~* **Done when:** You can answer “what’s slow, what broke, who’s affected” from a single dashboard.~~

~~### Feature flags & kill‑switches~~
~~* **Why:** When experimenting with new functionality you need the ability to disable it quickly without redeploying.~~ 
~~* **What to do:** Introduce a feature flag framework. Gate risky code paths (e.g. bulk AI calls, live streaming) and include toggles for read‑only mode and pausing ingestion.~~ 
~~* **Done when:** Any non‑core feature can be disabled in production with a configuration change.~~

~~### Zero‑downtime migrations~~
~~* **Why:** Schema changes can lock tables and cause downtime. An online migration strategy keeps the system available.~~ 
~~* **What to do:** Follow an “add → backfill → swap” pattern. First add new columns or tables, then backfill them in batches, finally switch the application to the new schema and drop unused columns. Use `NOT VALID` constraints promoted after backfill.~~
~~* **Done when:** Schema migrations complete in production without downtime or data corruption.~~

**P1 — Stability & performance**

~~### JOINed dashboard query~~
~~* **Why:** The dashboard currently issues multiple queries (N+1), slowing the first view. Joining these queries into one reduces round trips and leverages database optimisations.~~
~~* **What to do:** Write a single SQL statement that aggregates insights, companies and people. Create indexes to support the join predicates.~~ 
~~* **Done when:** The dashboard uses a single query and the p95 response time is under ~300 ms.~~

~~### L1/L2 caching~~
~~* **Why:** Serving every request from the database is slow and wastes resources. Caching recently accessed objects speeds up reads and protects the database.~~
~~* **What to do:** Implement an in‑process memory cache (L1) and a shared Redis cache (L2). Cache insights for 15 minutes, entities for one hour and metrics for five minutes. Write‑through and invalidate caches on write operations.~~
~~* **Done when:** The cache hit ratio exceeds 70 % and data remains consistent after writes.~~

~~### Batch RSS ingestion~~
~~* **Why:** Processing feeds item‑by‑item leads to high API costs and low throughput. Batching reduces overhead and deduplicates work.~~
~~* **What to do:** Fingerprint incoming items to drop duplicates, prioritize high‑value feeds, and send groups of items in a single AI call. Add metrics to monitor ingestion throughput and cost.~~
~~* **Done when:** Under load tests, throughput increases and the cost per item decreases.~~

~~### Service consolidation & dependency injection~~
~~* **Why:** Overlapping services (e.g. “Entity” vs. “Enrichment”) cause code duplication and inconsistent error handling. Dependency injection improves testability.~~
~~* **What to do:** Merge related services into a single layer per domain. Standardise error policies. Introduce a dependency injection container to construct services with explicit dependencies.~~
~~* **Done when:** Each domain has a single service implementation; errors are handled uniformly; services can be tested in isolation by injecting mocks.~~

~~### Core data model & migrations~~
~~* **Why:** Stable schemas are the foundation for entities, edges, and evidence.~~
~~* **What to do:** Add tables for `entity`, `entity_identifier`, `entity_attribute`, `entity_embedding`, `edge`, `edge_evidence`, and `event`. Enable `pgvector`. Create indexes for fast lookups and ANN search. Provide Drizzle migrations and rollback steps.~~
~~* **Done when:** Migrations run zero-downtime in dev/prod; basic CRUD works; p95 entity lookups <100ms and ANN queries <200ms on sample data.~~

~~### Entity canonicalization & identifiers~~
~~* **Why:** Consistent naming and strong IDs are critical for dedupe and joins.~~
~~* **What to do:** Implement canonical name builder (case/Unicode normalization, corp suffix stripping). Support identifiers (domain, CIK, Wikipedia slug, patent IDs). Enforce uniqueness per (scheme, value).~~
~~* **Done when:** Ingested entities store normalized names; duplicate inserts by ID collapse to one record; tests cover tricky name variants.~~

~~### Embeddings service (pgvector)~~
~~* **Why:** Semantic similarity powers fuzzy matching and discovery.~~
~~* **What to do:** Create a service to assemble entity text (name, description, key attributes), call embeddings, and store vectors in `entity_embedding`. Cache results in Redis with TTL and checksum invalidation.~~
~~* **Done when:** New/updated entities receive vectors within 2 minutes; ANN search returns relevant neighbors in top-K for test fixtures.~~




***


## TO DO:

**P2 — Code health & service boundaries**

### Outbox + Change Data Capture (CDC)
* **Why:** Updating the database and publishing events must be atomic to avoid lost updates. The transactional outbox pattern solves this problem.
* **What to do:** When a service modifies business entities, write a record into an outbox table in the same transaction. A separate relay process reads the table and publishes events to caches, search indexes or message brokers. Ensure idempotent consumers.
* **Done when:** All domain events are written to the outbox and relayed in order without loss.

### Read models & materialized views
* **Why:** Complex aggregations can be expensive to compute on the fly. Materialized views precompute results, reducing query latency.
* **What to do:** Identify hot queries (e.g. dashboard statistics) and create materialized views or pre‑aggregated tables. Schedule incremental refreshes and add indexes as needed.
* **Done when:** The dashboard queries the materialized view and returns results in under ~150 ms.

### Edge/API caching & ETags
* **Why:** Without HTTP caching, clients repeatedly fetch unchanged data. ETags and surrogate keys allow browsers and CDNs to cache responses.
* **What to do:** Add ETag headers to API responses and implement conditional GET/If‑None‑Match logic. Use surrogate keys to invalidate caches precisely when underlying data changes.
* **Done when:** Browsers and edge caches serve repeated reads without hitting the origin, and stale responses are never served.

**P3 — Intelligence & leverage**



### Deterministic entity resolution
* **Why:** High-precision dedupe prevents graph pollution.
* **What to do:** Write matching rules on identifiers and exact/near-exact names. Build a confidence score and auto-merge threshold. Provide merge operation that rewires edges and attributes with audit trail.
* **Done when:** Gold set of known duplicates auto-merge at ≥0.85 confidence; merges are idempotent and logged; no false merges on the test set.

### Fuzzy entity resolution (semantic + string)
* **Why:** Catch near-duplicates and aliases that lack strong IDs.
* **What to do:** Add Jaro-Winkler/Levenshtein checks + ANN candidate search. Combine into a weighted score; queue uncertain pairs for review.
* **Done when:** Recall improves by ≥20% over deterministic only; review queue shows top candidates with accept/decline; decisions persist.

### Relationship rules engine (deterministic edges)
* **Why:** Obvious, high-confidence links should appear instantly.
* **What to do:** Implement rules for `employee_of`, `worked_together` (overlap ≥3 months), `assignee_of`, `co_inventor`, and `same_market` (exact category match). Compute strength and record evidence rows.
* **Done when:** Sample fixtures produce correct edges with strengths; hover text renders from templates; evidence points to source rows/URLs.

### Statistical relationships (co-mention uplift)
* **Why:** Reveal non-obvious ties driven by recent data.
* **What to do:** From `event` table, compute co-mention counts over sliding windows and compare to baseline (PMI-style). Create/boost edges when uplift exceeds a threshold and there are ≥3 distinct sources.
* **Done when:** Co-mentioned entities gain edges with confidence proportional to uplift; false positives drop when the threshold changes.

### Embedding-based relatedness (soft edges)
* **Why:** Suggest exploratory connections without over-claiming.
* **What to do:** Add cosine-similarity checks and require at least one shared soft attribute (industry or region). Label as `related_domain`; hide behind an “Explore” toggle in UI.
* **Done when:** Toggle shows dotted edges with explanations; precision improves when soft-attribute guard is on.

### Time-decay & freshness
* **Why:** Stale relationships should fade, fresh ones should pop.
* **What to do:** Nightly job reduces `strength`/`confidence` on edges lacking new evidence; new events refresh `last_seen_at` and reverse decay.
* **Done when:** Edges age visibly over weeks in a sandbox dataset; adding new events lifts confidence above display threshold again.

### Insight generation (templated)
* **Why:** Users need clear, sourced one-liners on hover.
* **What to do:** Create deterministic templates per edge type (employment overlap, patents, assignee). Include dates, counts, and source labels. Cache rendered strings with ETag on top evidence.
* **Done when:** Hover popovers render in ≤50ms from cache; template unit tests verify phrasing and variable binding.

### Insight generation (LLM-phrased long tail)
* **Why:** Some edges need natural language that templates can’t cover.
* **What to do:** Build a guarded LLM call that summarizes only from provided evidence snippets. Reject generations that introduce unseen facts; fall back to a safe template.
* **Done when:** Random audits show zero hallucinations; latency <1.5s p95 with caching; cost per 1k insights stays within budget.

### Board drop → fast neighbor match
* **Why:** The “magic” is connections appearing as soon as a card hits the canvas.
* **What to do:** On card add, run a lightweight neighbor search (blocking filters + ANN top-K) against entities on the board (plus small halo). Compute edges and push via WebSocket.
* **Done when:** Edge lines appear within 500ms–1.5s after drop on test boards of 50–500 nodes; no UI thread blocking.

### WebSocket event gateway
* **Why:** Real-time updates keep boards alive without refresh.
* **What to do:** Add WS channels per board. Emit `edge.added/updated/removed` and minimal payloads. Backoff/retry with client resubscribe.
* **Done when:** Lost connections recover; message order is preserved per board; soak tests confirm stability under 1k events/min.

### BullMQ job system & scheduling
* **Why:** Async work separates UX from heavy compute.
* **What to do:** Create queues for `entity:embed`, `entity:resolve`, `relations:build`, `relations:matchNeighbors`, `insight:generate`, `maintenance:decayEdges`, and `ingest:*`. Add concurrency controls and dead-letter queues.
* **Done when:** Jobs are observable in a dashboard; retries and DLQs work; no starvation under mixed workloads.

### Events ingestion from RSS into `event`
* **Why:** Fresh news drives new edges and insights.
* **What to do:** Extend existing RSS pipeline to map items to `event` rows with `entityId` linkage (via NER and rules). Store title, summary, URL, publishedAt, and source.
* **Done when:** New items appear in `event` with correct entity mapping ≥90% on test feeds; bad items are quarantined with reasons.

### Caching strategy (Redis)
* **Why:** Keep latency and costs predictable.
* **What to do:** Cache embeddings, candidate neighbor sets, and rendered insights with sensible TTLs (24h embeddings, 1–6h candidates/insights). Invalidate on entity/attribute/evidence changes.
* **Done when:** Cache hit rate ≥70% on hover/neighbor calls; API p95 improves by ≥30% vs cold.

### REST API for boards, entities & edges
* **Why:** Frontend and external systems need stable contracts.
* **What to do:** Implement endpoints: `POST /entities`, `GET /entities/:id`, `GET /entities/search`, `POST /boards/:id/cards`, `GET /boards/:id/edges`, `POST /edges/recompute`, `GET /edges/:id/insight`.
* **Done when:** OpenAPI schema passes validation; integration tests cover happy paths and errors; rate limits applied per tenant.

### Graph rendering & thresholds (UI)
* **Why:** Control noise vs signal and make exploration safe.
* **What to do:** Integrate Cytoscape.js. Add “Display threshold” slider and “Explore” toggle for soft edges. Dotted styling for suggestions; tooltips show insights and sources.
* **Done when:** Users can filter edges live without reloading; FPS stays ≥45 on a 500-node board; accessibility labels work.

### Feedback loop on edges & merges
* **Why:** Humans improve precision and training data.
* **What to do:** Add UI controls to mark an edge “incorrect” or “useful,” and to approve/decline merge suggestions. Record feedback for thresholds and future models.
* **Done when:** Feedback writes to DB; nightly job updates thresholds/weights; precision improves on evaluation set after applying feedback.

### Observability & structured logging
* **Why:** Diagnose performance, quality, and cost issues.
* **What to do:** Add JSON logs (pino/winston), request IDs, and job IDs. Track metrics: edges created per type/day, average confidence, false-positive rate, queue latency. Log to file with rotation; expose `/healthz` and `/metrics`.
* **Done when:** Dashboards show SLOs; red/yellow alerts fire on queue backlog and error spikes; logs survive restarts and can be downloaded.

### Cost controls & model routing
* **Why:** Keep LLM spend predictable.
* **What to do:** Track per-operation cost; route only long-tail insights to LLM; short-circuit high-confidence template cases. Enforce daily budget caps with graceful degradation.
* **Done when:** Budget cap halts LLM calls but keeps templates; cost report shows per-day totals and savings; no user-visible failures.

### Security, RBAC & tenant isolation
* **Why:** Protect data and comply with enterprise expectations.
* **What to do:** Scope all reads/writes by tenant; implement role-based access (admin/user); enforce rate limits per tenant; validate input via Zod. Mask secrets in logs.
* **Done when:** Automated tests prove tenant isolation; permission checks block cross-tenant access; rate limiter metrics show fair usage.

### Multi-tenant, API-first & mobile-ready
* **Why:** Enterprise and mobile users require isolation, public APIs, and offline support.
* **What to do:** Scope data by organisation, implement role-based access control, and add quotas per tenant. Expose REST/GraphQL APIs and webhooks. Build a progressive web app with offline sync and push notifications.
* **Done when:** Automated tests confirm tenant isolation, the API covers key operations, and the mobile app works offline and installs on devices.

### Admin tools & backfills
* **Why:** Operate safely and fix past data.
* **What to do:** Add admin routes to trigger recompute/backfill for embeddings, edges, and insights; progress reporting; cancel/rollback support.
* **Done when:** Backfills run chunked with live progress; can be paused/resumed; average throughput meets targets without DB timeouts.

### Naming & copy polish for the feature
* **Why:** A crisp name and microcopy drive adoption.
* **What to do:** Choose a brandable name (e.g., Mesh, Lattice, Nexus, Synapse, Constellation). Update UI labels, tooltips, and empty-state copy to explain value in one line.
* **Done when:** Name is consistently used in UI/docs; empty state shows a 1–2 sentence pitch; user tests show comprehension in <10 seconds.

### Workflow automation engine
* **Why:** Insights should trigger actions (e.g. Slack alerts, dossier updates) automatically. A workflow engine turns passive information into active operations.
* **What to do:** Implement a simple trigger‑action engine (similar to n8n). Build at least three flows: RSS to Slack, entity tracking alerts, and market shift notifications. Use feature flags to enable/disable flows.
* **Done when:** Non‑technical users can create and manage workflows, and the three built‑in flows run end‑to‑end.

**P4 — Scale & future‑proofing**

### Cost‑at‑scale tactics
* **Why:** As usage grows, costs can balloon. Controlling resource consumption keeps the project sustainable.
* **What to do:** Continue batching and deduplicating work, route low‑priority jobs to cheaper AI models, fingerprint content to avoid reprocessing, and implement dynamic budget monitoring.
* **Done when:** Cost per user stays within defined targets under representative load.

### Integration & modularity hardening
* **Why:** Clear service boundaries and plugin interfaces make it easy to add new data sources or features without touching core logic.
* **What to do:** Define microservice boundaries explicitly. Create a plugin interface for adding new data sources. Implement WebSocket channels for live insights.
* **Done when:** A new data source can be integrated via a plugin and live updates stream reliably to clients.

### Multi‑tenant, API‑first & mobile‑ready
* **Why:** Enterprise and mobile users require isolation, public APIs, and offline support.
* **What to do:** Scope data by organisation, implement role‑based access control, and add quotas per tenant. Expose REST/GraphQL APIs and webhooks. Build a progressive web app with offline sync and push notifications.
* **Done when:** Automated tests confirm tenant isolation, the API covers key operations, and the mobile app works offline and installs on devices.

---

This backlog is a living document. As we complete items or discover new requirements, we will update this file accordingly. For an understanding of individual concepts referenced here and how they're being used in Prismo (idempotency, connection pooling, caching, etc.), see the deep‑dive pages.
