# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: CDN & Edge Configuration Manager · Created: 2026-05-20

## Philosophy

This model treats every configuration change as an immutable event in an append-only event store. The event store is the single source of truth; all current-state views (the "active configuration," provider sync status, cost summaries) are materialized projections rebuilt from the event stream. This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from purpose-built projections.

This architecture is particularly well-suited to CDN configuration management because the domain has strong natural requirements for full audit trails ("who changed this cache rule and when?"), temporal queries ("what was the active configuration at 3:47 AM when the incident began?"), rollback capability ("replay events up to version N to reconstruct a known-good state"), and multi-provider synchronization ("which provider has acknowledged which configuration events?"). Akamai's immutable property versioning and Cloudflare's zone versioning (April 2026) already hint at event-sourced thinking in the CDN space.

The trade-off is operational complexity. Event replay, projection rebuilding, eventual consistency between write and read models, and event schema evolution all require infrastructure that a simple CRUD application does not. However, for a platform where configuration changes propagate to global edge infrastructure and mistakes can expose private data or cause outages, the ability to replay, audit, and reason about every change is a significant safety net.

**Best for:** Organizations that require complete audit trails, temporal querying, point-in-time reconstruction, and the ability to replay configuration history for debugging or compliance.

**Trade-offs:**
- Pro: Complete, immutable audit trail of every configuration change
- Pro: Point-in-time reconstruction ("what was active at timestamp X?")
- Pro: Natural rollback: replay events up to a known-good point
- Pro: Decoupled read models can be optimized independently for different query patterns
- Con: Eventual consistency between event store and projections
- Con: Event schema evolution requires versioning and migration strategies
- Con: Higher operational complexity (event replay, projection rebuilding, compaction)
- Con: Simple "show me the current config" queries require reading from projections, not the source of truth directly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 9111 (HTTP Caching) | Cache directive values stored as event payloads; projections materialize current directive state |
| RFC 9213 (CDN-Cache-Control) | Targeted cache control changes modeled as `CdnCacheDirectiveSet` events |
| RFC 8555 (ACME) | Certificate lifecycle transitions modeled as events (OrderCreated, ChallengeCompleted, CertificateIssued, CertificateExpiring) |
| OpenTelemetry | Event metadata includes trace context (trace_id, span_id) for correlating config changes with observability pipelines |
| ISO 3166-1 | Geographic routing changes reference ISO country/region codes in event payloads |
| OCSF (Open Cybersecurity Schema Framework) | Audit events follow OCSF activity categories for security event interoperability |

---

## Event Store (Source of Truth)

```sql
-- The single append-only event store. This is the ONLY table that receives writes
-- for configuration state. All other tables are projections rebuilt from this stream.
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL, -- aggregate root ID (property, organization, etc.)
    stream_type     VARCHAR(100) NOT NULL,
    -- e.g., 'Property', 'Organization', 'TrafficPolicy', 'Certificate'
    event_type      VARCHAR(200) NOT NULL,
    -- e.g., 'CacheRuleCreated', 'ConfigVersionActivated', 'TrafficWeightChanged'
    event_version   INTEGER NOT NULL, -- schema version of this event type
    sequence_number BIGINT NOT NULL, -- monotonically increasing per stream
    payload         JSONB NOT NULL, -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata contains: user_id, ip_address, trace_id, span_id, correlation_id
    caused_by       UUID, -- event_id that triggered this event (for sagas)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, sequence_number)
);

CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);
CREATE INDEX idx_events_stream_type ON events(stream_type);

-- Optimistic concurrency: before appending, check that the expected sequence_number
-- matches the current max for the stream.
```

### Event Type Catalog

```sql
-- Registry of known event types with their JSON schemas for validation
CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    event_version   INTEGER NOT NULL,
    description     TEXT,
    payload_schema  JSONB NOT NULL, -- JSON Schema (Draft 2020-12) for payload validation
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    superseded_by   VARCHAR(200),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Event Payloads

```jsonc
// Event: CacheRuleCreated
{
  "event_type": "CacheRuleCreated",
  "stream_type": "Property",
  "stream_id": "550e8400-e29b-41d4-a716-446655440000",
  "payload": {
    "rule_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "config_version_number": 12,
    "name": "Cache static assets 30 days",
    "priority": 10,
    "match": {
      "path_pattern": "/static/*",
      "path_type": "glob",
      "methods": ["GET", "HEAD"],
      "content_types": ["image/*", "text/css", "application/javascript"]
    },
    "directives": {
      "cache_action": "cache",
      "max_age_seconds": 2592000,
      "s_maxage_seconds": 2592000,
      "stale_while_revalidate_seconds": 86400,
      "stale_if_error_seconds": 604800,
      "cdn_cache_control_max_age": 2592000,
      "surrogate_key": "static-assets"
    }
  },
  "metadata": {
    "user_id": "user-uuid",
    "ip_address": "203.0.113.42",
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "correlation_id": "deploy-batch-2026-05-20-001"
  }
}

// Event: ConfigVersionActivated
{
  "event_type": "ConfigVersionActivated",
  "stream_type": "Property",
  "stream_id": "550e8400-e29b-41d4-a716-446655440000",
  "payload": {
    "version_number": 12,
    "previous_active_version": 11,
    "environment": "production",
    "activation_mode": "immediate"
  },
  "metadata": {
    "user_id": "user-uuid",
    "approval_id": "approval-uuid",
    "ip_address": "203.0.113.42"
  }
}

// Event: TrafficWeightChanged
{
  "event_type": "TrafficWeightChanged",
  "stream_type": "TrafficPolicy",
  "stream_id": "policy-uuid",
  "payload": {
    "provider_slug": "cloudflare",
    "previous_weight": 60,
    "new_weight": 40,
    "reason": "ml_optimization",
    "performance_score": 0.87
  }
}

// Event: ProviderPurgeCompleted
{
  "event_type": "ProviderPurgeCompleted",
  "stream_type": "Property",
  "stream_id": "property-uuid",
  "payload": {
    "purge_request_id": "purge-uuid",
    "provider_slug": "fastly",
    "purge_type": "surrogate_key",
    "purge_target": "static-assets",
    "propagation_ms": 142,
    "provider_purge_id": "fastly-purge-12345"
  }
}

// Event: CertificateRenewed (ACME lifecycle)
{
  "event_type": "CertificateRenewed",
  "stream_type": "Certificate",
  "stream_id": "cert-uuid",
  "payload": {
    "domain_names": ["cdn.example.com", "*.cdn.example.com"],
    "issuer": "letsencrypt",
    "previous_expires_at": "2026-08-15T00:00:00Z",
    "new_expires_at": "2026-11-13T00:00:00Z",
    "acme_order_id": "order-12345"
  }
}
```

---

## Read Model Projections

These tables are rebuilt from the event stream. They can be dropped and reconstructed at any time by replaying events.

### Current Configuration Projection

```sql
-- Materialized view of current active configuration per property
CREATE TABLE projection_active_configs (
    property_id         UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    property_name       VARCHAR(255) NOT NULL,
    property_slug       VARCHAR(100) NOT NULL,
    active_version      INTEGER,
    environment         VARCHAR(50),
    activated_at        TIMESTAMPTZ,
    activated_by        UUID,
    last_event_id       UUID NOT NULL, -- watermark: last event processed
    last_event_at       TIMESTAMPTZ NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Current cache rules for active version
CREATE TABLE projection_cache_rules (
    rule_id             UUID PRIMARY KEY,
    property_id         UUID NOT NULL,
    config_version      INTEGER NOT NULL,
    name                VARCHAR(255) NOT NULL,
    priority            INTEGER NOT NULL,
    is_enabled          BOOLEAN NOT NULL,
    match_criteria      JSONB NOT NULL,
    cache_directives    JSONB NOT NULL,
    -- Flattened for common queries:
    max_age_seconds     INTEGER,
    surrogate_key       VARCHAR(500),
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_cache_rules_property ON projection_cache_rules(property_id);
```

### Provider Sync Projection

```sql
-- Current sync status per property per provider
CREATE TABLE projection_provider_sync (
    property_id         UUID NOT NULL,
    cdn_provider_slug   VARCHAR(50) NOT NULL,
    sync_status         VARCHAR(50) NOT NULL,
    provider_resource_id VARCHAR(500),
    provider_version    VARCHAR(255),
    last_synced_at      TIMESTAMPTZ,
    last_sync_event_id  UUID,
    pending_events      INTEGER NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, cdn_provider_slug)
);
```

### Traffic Routing Projection

```sql
CREATE TABLE projection_traffic_weights (
    policy_id           UUID NOT NULL,
    property_id         UUID NOT NULL,
    cdn_provider_slug   VARCHAR(50) NOT NULL,
    current_weight      INTEGER NOT NULL,
    strategy            VARCHAR(50) NOT NULL,
    last_change_reason  VARCHAR(100),
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (policy_id, cdn_provider_slug)
);

CREATE INDEX idx_proj_traffic_property ON projection_traffic_weights(property_id);
```

### Cost Summary Projection

```sql
CREATE TABLE projection_cost_summary (
    organization_id     UUID NOT NULL,
    cdn_provider_slug   VARCHAR(50) NOT NULL,
    period_month        DATE NOT NULL, -- first day of month
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    total_requests      BIGINT NOT NULL DEFAULT 0,
    total_cost_usd      NUMERIC(12, 4) NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, cdn_provider_slug, period_month)
);
```

### Temporal Query Projection

```sql
-- Snapshot table for point-in-time queries
-- Periodically populated by a background process that captures full config state
CREATE TABLE projection_config_snapshots (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id         UUID NOT NULL,
    snapshot_at         TIMESTAMPTZ NOT NULL,
    config_state        JSONB NOT NULL, -- full serialized configuration at this point
    event_id            UUID NOT NULL, -- event that triggered this snapshot
    sequence_number     BIGINT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshots_property_time ON projection_config_snapshots(property_id, snapshot_at);
```

---

## Supporting Infrastructure Tables

These are NOT projections; they store data that doesn't naturally fit the event model.

```sql
-- Reference data: CDN provider capabilities (slowly changing, admin-managed)
CREATE TABLE cdn_providers (
    slug            VARCHAR(50) PRIMARY KEY,
    display_name    VARCHAR(100) NOT NULL,
    api_base_url    VARCHAR(500) NOT NULL,
    auth_method     VARCHAR(50) NOT NULL,
    supports_http3  BOOLEAN NOT NULL DEFAULT false,
    supports_esi    BOOLEAN NOT NULL DEFAULT false,
    purge_propagation_ms INTEGER,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- User identity (managed by auth system, not event-sourced)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Subscription/checkpoint tracking for projection processors
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'paused', 'rebuilding', 'failed')),
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Temporal Queries

```sql
-- "What was the active configuration at 3:47 AM during the incident?"
-- Option A: From snapshots (fast)
SELECT config_state
FROM projection_config_snapshots
WHERE property_id = $1
  AND snapshot_at <= '2026-05-20T03:47:00Z'
ORDER BY snapshot_at DESC
LIMIT 1;

-- Option B: From event replay (authoritative)
SELECT *
FROM events
WHERE stream_id = $1
  AND stream_type = 'Property'
  AND created_at <= '2026-05-20T03:47:00Z'
ORDER BY sequence_number ASC;
-- Application replays these events to reconstruct state

-- "Show me all cache rule changes in the last 24 hours across all properties"
SELECT e.stream_id AS property_id,
       e.event_type,
       e.payload->>'name' AS rule_name,
       e.payload->'directives'->>'max_age_seconds' AS max_age,
       e.metadata->>'user_id' AS changed_by,
       e.created_at
FROM events e
WHERE e.event_type IN ('CacheRuleCreated', 'CacheRuleUpdated', 'CacheRuleDeleted')
  AND e.created_at >= now() - INTERVAL '24 hours'
ORDER BY e.created_at DESC;

-- "Who changed the traffic weights for Cloudflare in the last week?"
SELECT e.payload->>'previous_weight' AS old_weight,
       e.payload->>'new_weight' AS new_weight,
       e.payload->>'reason' AS reason,
       e.metadata->>'user_id' AS user_id,
       e.created_at
FROM events e
WHERE e.event_type = 'TrafficWeightChanged'
  AND e.payload->>'provider_slug' = 'cloudflare'
  AND e.created_at >= now() - INTERVAL '7 days'
ORDER BY e.created_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | Core event table + event type registry |
| Projections (Read Models) | 6 | Active config, cache rules, provider sync, traffic, cost, snapshots |
| Reference Data | 2 | CDN providers, users |
| Infrastructure | 1 | Projection checkpoints |
| **Total** | **11** | Far fewer tables; complexity is in event payloads and projection logic |

---

## Key Design Decisions

1. **Single event table rather than per-aggregate tables.** All events for all aggregate types (Property, TrafficPolicy, Certificate, Organization) go into one `events` table, partitioned by `stream_type` if volume demands it. This simplifies event ordering and cross-aggregate queries.

2. **JSONB payloads with JSON Schema validation.** Event payloads are JSONB for flexibility, but every event type has a registered JSON Schema (Draft 2020-12) in `event_type_registry` for validation before insertion. This gives the flexibility of schema-on-read with the safety of validated writes.

3. **Optimistic concurrency via sequence_number.** Before appending an event, the writer checks that the expected sequence number matches the current max for the stream. This prevents conflicting concurrent modifications without database-level locks.

4. **Projections are disposable.** Every projection table can be dropped and rebuilt by replaying the event stream from the beginning (or from the last snapshot). The `projection_checkpoints` table tracks each projection processor's position in the event stream.

5. **Periodic snapshots for fast temporal queries.** Rather than replaying potentially millions of events for a point-in-time query, the system periodically captures full configuration state as JSONB snapshots. These serve as fast-path lookups for incident investigation.

6. **Event metadata includes OpenTelemetry trace context.** Every event carries `trace_id` and `span_id` in its metadata, enabling correlation between configuration changes and distributed traces from the observability pipeline.

7. **Caused-by chaining for sagas.** Multi-step operations (e.g., "activate config version" triggers "sync to Cloudflare" + "sync to Fastly" + "invalidate cache") are linked via the `caused_by` field, creating a causal chain that can be traversed for debugging.

8. **Event versioning for schema evolution.** The `event_version` field on each event allows the system to handle schema changes gracefully. A projection processor knows how to handle v1 and v2 of `CacheRuleCreated` differently, and the `event_type_registry` tracks which versions are current vs. deprecated.
