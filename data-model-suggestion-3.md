# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: CDN & Edge Configuration Manager · Created: 2026-05-20

## Philosophy

This model keeps the structural backbone relational (organizations, properties, domains, provider credentials) while storing provider-specific configuration, match criteria, and cache directives in JSONB columns. The core insight is that CDN providers differ wildly in their configuration surfaces — Akamai has Property Manager behaviors/criteria, Fastly has VCL objects and conditions, Cloudflare has rulesets with expressions — and trying to normalize all of these into shared relational columns produces either an explosion of provider-specific tables or a lowest-common-denominator abstraction that loses important provider features.

The hybrid approach solves this by defining a portable "abstract configuration" layer in JSONB (the platform's own cache rule format) alongside a "provider-native configuration" JSONB column that stores the exact provider-specific representation. When a user creates a cache rule, the platform stores both the abstract form (for cross-provider display and AI analysis) and the provider-native form (for accurate sync and drift detection). This mirrors how Terraform stores both its own resource state and the provider's raw API response.

This architecture is ideal for rapid iteration and multi-provider support. Adding a new CDN provider doesn't require schema migrations — only new JSONB structures in the provider-native columns. The trade-off is that JSONB queries require specialized operators (@>, ?, jsonb_path_query) and GIN indexes, and the database cannot enforce referential integrity within JSONB structures.

**Best for:** Teams prioritizing rapid multi-provider integration, MVP velocity, and tolerance for varied provider-specific configuration shapes without schema migrations.

**Trade-offs:**
- Pro: No schema migrations when adding new CDN providers or provider-specific features
- Pro: Stores both abstract and provider-native config representations naturally
- Pro: Rapid MVP development; fewer tables to manage
- Pro: JSONB GIN indexes enable fast containment queries
- Con: No foreign key constraints within JSONB; integrity enforced at application layer
- Con: JSONB queries less readable than simple column comparisons
- Con: Harder to do cross-provider aggregation queries on deeply nested JSONB fields
- Con: Risk of schema drift within JSONB if validation is not enforced

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 9111 (HTTP Caching) | Abstract cache directives stored as a well-defined JSONB structure with RFC 9111 field names |
| RFC 9213 (CDN-Cache-Control) | CDN-targeted directives stored in the `targeted_directives` JSONB key |
| RFC 9114 / RFC 9000 (HTTP/3 / QUIC) | Protocol settings stored in `protocol_config` JSONB on domains |
| RFC 8555 (ACME) | Certificate ACME state stored as JSONB lifecycle document |
| JSON Schema (Draft 2020-12) | Every JSONB column has a registered JSON Schema for application-level validation |
| ISO 3166-1 | Geographic rules reference ISO country codes within JSONB match criteria |
| HCL / Terraform State | Provider-native configs structurally compatible with Terraform resource attributes |

---

## Core Platform Tables (Relational)

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {"default_ttl": 86400, "max_rules_per_version": 100,
    --                     "allowed_providers": ["cloudflare", "fastly"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    preferences     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example: ["config.read", "config.write", "purge.execute", "traffic.steer"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_members(organization_id);
```

### Provider Credentials

```sql
CREATE TABLE provider_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    provider_slug   VARCHAR(50) NOT NULL,
    -- 'cloudflare', 'fastly', 'akamai', 'aws_cloudfront', 'gcore', 'bunnycdn'
    display_name    VARCHAR(255) NOT NULL,
    auth_config     JSONB NOT NULL,
    -- auth_config varies by provider:
    -- Cloudflare: {"method": "bearer_token", "token_ref": "vault://..."}
    -- Akamai:     {"method": "edgegrid", "client_token": "...", "access_token": "...",
    --              "client_secret_ref": "vault://...", "host": "akab-xxx.luna.akamaiapis.net"}
    -- AWS:        {"method": "sigv4", "access_key_ref": "vault://...", "region": "us-east-1"}
    -- Fastly:     {"method": "api_key", "key_ref": "vault://..."}
    provider_capabilities JSONB NOT NULL DEFAULT '{}',
    -- Discovered after connection: {"supports_http3": true, "supports_esi": false,
    --   "purge_methods": ["url", "surrogate_key", "all"], "purge_propagation_ms": 150}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_validated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, provider_slug, display_name)
);

CREATE INDEX idx_provider_conn_org ON provider_connections(organization_id);
CREATE INDEX idx_provider_conn_slug ON provider_connections(provider_slug);
```

### Properties & Domains

```sql
CREATE TABLE properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    tags            JSONB NOT NULL DEFAULT '[]',
    -- tags example: ["production", "media", "us-east"]
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"team": "platform-engineering", "cost_center": "CC-1234"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, slug)
);

CREATE INDEX idx_properties_org ON properties(organization_id);
CREATE INDEX idx_properties_tags ON properties USING GIN (tags);

CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    hostname        VARCHAR(500) NOT NULL UNIQUE,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    protocol_config JSONB NOT NULL DEFAULT '{}',
    -- protocol_config example:
    -- {"tls": {"enabled": true, "min_version": "1.2", "cipher_suites": ["TLS_AES_128_GCM_SHA256"]},
    --  "http3": {"enabled": true, "quic": {"connection_migration": true, "zero_rtt": false}},
    --  "http2": {"enabled": true, "server_push": false}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_domains_property ON domains(property_id);

CREATE TABLE origins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    hostname        VARCHAR(500) NOT NULL,
    port            INTEGER NOT NULL DEFAULT 443,
    protocol        VARCHAR(10) NOT NULL DEFAULT 'https',
    health_check    JSONB NOT NULL DEFAULT '{}',
    -- health_check example:
    -- {"path": "/health", "interval_sec": 30, "timeout_ms": 5000,
    --  "healthy_threshold": 3, "unhealthy_threshold": 2, "expected_status": [200]}
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- connection_config example:
    -- {"connect_timeout_ms": 5000, "read_timeout_ms": 30000,
    --  "keepalive_connections": 64, "retry_count": 2}
    weight          INTEGER NOT NULL DEFAULT 100,
    is_backup       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_origins_property ON origins(property_id);
```

---

## Configuration Versioning

```sql
CREATE TABLE config_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'validating', 'staged', 'active', 'superseded', 'rollback')),
    description     TEXT,
    created_by      UUID REFERENCES users(id),
    activated_at    TIMESTAMPTZ,
    activated_by    UUID REFERENCES users(id),
    parent_version_id UUID REFERENCES config_versions(id),
    validation_results JSONB,
    -- validation_results example:
    -- {"valid": false, "errors": [{"rule_id": "...", "code": "OVERLAP", "message": "..."}],
    --  "warnings": [{"code": "STALE_TTL_HIGH", "message": "stale-while-revalidate > 30 days"}]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(property_id, version_number)
);

CREATE INDEX idx_config_versions_property ON config_versions(property_id);
CREATE INDEX idx_config_versions_status ON config_versions(status);
```

---

## Edge Configuration Rules (Hybrid JSONB)

```sql
CREATE TABLE edge_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_version_id UUID NOT NULL REFERENCES config_versions(id) ON DELETE CASCADE,
    rule_type       VARCHAR(50) NOT NULL
                    CHECK (rule_type IN ('cache', 'header', 'redirect', 'rewrite',
                                         'rate_limit', 'waf', 'geo_block', 'esi')),
    name            VARCHAR(255) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,

    -- Abstract match criteria (portable across providers)
    match_criteria  JSONB NOT NULL DEFAULT '{}',
    -- match_criteria examples:
    -- Simple: {"path": {"pattern": "/static/*", "type": "glob"}, "methods": ["GET", "HEAD"]}
    -- Complex: {"and": [
    --   {"path": {"pattern": "/api/v2/*", "type": "glob"}},
    --   {"header": {"name": "Accept", "contains": "application/json"}},
    --   {"not": {"geo": {"countries": ["CN", "RU"]}}}
    -- ]}

    -- Abstract action/directives (portable across providers)
    action          JSONB NOT NULL,
    -- action for cache rule:
    -- {"type": "cache", "directives": {
    --   "max_age": 2592000, "s_maxage": 2592000,
    --   "stale_while_revalidate": 86400, "stale_if_error": 604800,
    --   "cdn_cache_control": {"max_age": 2592000},
    --   "surrogate_key": "static-assets",
    --   "serve_stale_on_error": true, "respect_origin_headers": true
    -- }}
    --
    -- action for header rule:
    -- {"type": "set_header", "headers": [
    --   {"name": "X-Frame-Options", "value": "DENY", "target": "response"},
    --   {"name": "X-Content-Type-Options", "value": "nosniff", "target": "response"}
    -- ]}
    --
    -- action for WAF rule:
    -- {"type": "waf", "mode": "block", "owasp_categories": ["A01", "A03"],
    --  "sensitivity": "medium", "custom_rules": [...]}

    -- AI analysis metadata (populated by ML pipeline)
    ai_analysis     JSONB,
    -- ai_analysis example:
    -- {"recommended_ttl": 604800, "cache_hit_prediction": 0.94,
    --  "risk_score": 0.12, "similar_rules": ["rule-uuid-1", "rule-uuid-2"],
    --  "optimization_suggestions": ["Consider adding stale-if-error for resilience"]}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edge_rules_version ON edge_rules(config_version_id);
CREATE INDEX idx_edge_rules_type ON edge_rules(rule_type);
CREATE INDEX idx_edge_rules_priority ON edge_rules(config_version_id, priority);
CREATE INDEX idx_edge_rules_match ON edge_rules USING GIN (match_criteria);
CREATE INDEX idx_edge_rules_action ON edge_rules USING GIN (action);
```

### Provider-Native Configurations

```sql
-- Stores the provider-specific representation of each edge rule.
-- One row per rule per provider. Generated by the translation engine.
CREATE TABLE provider_rule_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_rule_id    UUID NOT NULL REFERENCES edge_rules(id) ON DELETE CASCADE,
    provider_slug   VARCHAR(50) NOT NULL,
    connection_id   UUID NOT NULL REFERENCES provider_connections(id),

    -- The provider-native representation of this rule
    native_config   JSONB NOT NULL,
    -- Cloudflare native_config example:
    -- {"ruleset_phase": "http_request_cache_settings",
    --  "expression": "(http.request.uri.path wildcard \"/static/*\")",
    --  "action": "set_cache_settings",
    --  "action_parameters": {"cache": true, "edge_ttl": {"mode": "override_origin", "default": 2592000},
    --                        "browser_ttl": {"mode": "override_origin", "default": 2592000}}}
    --
    -- Fastly native_config example:
    -- {"vcl_snippet": "if (req.url ~ \"^/static/\") {\n  set beresp.ttl = 30d;\n  ...\n}",
    --  "condition": {"type": "REQUEST", "statement": "req.url ~ \"^/static/\"", "priority": 10},
    --  "cache_setting": {"action": "cache", "ttl": 2592000, "stale_ttl": 86400}}
    --
    -- Akamai native_config example:
    -- {"behaviors": [{"name": "caching", "options": {"behavior": "MAX_AGE", "mustRevalidate": false,
    --    "defaultTtl": "30d"}}],
    --  "criteria": [{"name": "path", "options": {"matchOperator": "MATCHES_ONE_OF",
    --    "values": ["/static/*"]}}]}

    -- Sync state for this specific rule at this provider
    sync_status     VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (sync_status IN ('pending', 'syncing', 'synced', 'drifted', 'failed', 'not_supported')),
    provider_resource_id VARCHAR(500), -- e.g., Cloudflare ruleset rule ID
    last_synced_at  TIMESTAMPTZ,
    drift_diff      JSONB, -- JSON diff between expected and actual provider state
    error_message   TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(edge_rule_id, provider_slug)
);

CREATE INDEX idx_provider_rules_rule ON provider_rule_configs(edge_rule_id);
CREATE INDEX idx_provider_rules_slug ON provider_rule_configs(provider_slug);
CREATE INDEX idx_provider_rules_sync ON provider_rule_configs(sync_status);
CREATE INDEX idx_provider_rules_native ON provider_rule_configs USING GIN (native_config);
```

---

## Traffic Steering

```sql
CREATE TABLE traffic_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    config_version_id UUID NOT NULL REFERENCES config_versions(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    strategy        VARCHAR(50) NOT NULL DEFAULT 'weighted',
    is_enabled      BOOLEAN NOT NULL DEFAULT true,

    -- Strategy-specific configuration in JSONB
    strategy_config JSONB NOT NULL DEFAULT '{}',
    -- weighted: {"normalize": true}
    -- geolocation: {"default_provider": "cloudflare", "fallback_provider": "fastly"}
    -- latency: {"measurement_window_minutes": 5, "min_sample_size": 100}
    -- cost: {"budget_usd_monthly": 50000, "optimize_for": "cost_per_gb"}
    -- ml_optimized: {"model_id": "traffic-router-v3", "constraints": {"max_single_provider_pct": 80}}

    -- Provider targets with weights and geo overrides
    targets         JSONB NOT NULL,
    -- targets example:
    -- [
    --   {"provider_slug": "cloudflare", "connection_id": "uuid", "weight": 60,
    --    "max_cost_per_gb_usd": 0.02, "enabled": true},
    --   {"provider_slug": "fastly", "connection_id": "uuid", "weight": 30,
    --    "max_cost_per_gb_usd": 0.04, "enabled": true},
    --   {"provider_slug": "bunnycdn", "connection_id": "uuid", "weight": 10,
    --    "max_cost_per_gb_usd": 0.005, "enabled": true}
    -- ]

    -- Geographic overrides
    geo_overrides   JSONB NOT NULL DEFAULT '[]',
    -- geo_overrides example:
    -- [
    --   {"countries": ["JP", "KR", "SG"], "provider_slug": "gcore", "weight": 100},
    --   {"countries": ["DE", "FR", "GB"], "provider_slug": "cloudflare", "weight": 70,
    --    "fallback": "fastly"},
    --   {"continents": ["AF"], "provider_slug": "bunnycdn", "weight": 100}
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_traffic_policies_property ON traffic_policies(property_id);
CREATE INDEX idx_traffic_targets ON traffic_policies USING GIN (targets);
CREATE INDEX idx_traffic_geo ON traffic_policies USING GIN (geo_overrides);
```

---

## Certificates & TLS

```sql
CREATE TABLE certificates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    domain_names    VARCHAR(500)[] NOT NULL,
    issuer          VARCHAR(255),

    -- ACME lifecycle as JSONB document
    acme_state      JSONB,
    -- acme_state example:
    -- {"order_url": "https://acme-v02.api.letsencrypt.org/acme/order/12345",
    --  "status": "valid",
    --  "challenges": [{"type": "http-01", "status": "valid", "validated_at": "..."},
    --                 {"type": "dns-01", "status": "valid", "validated_at": "..."}],
    --  "finalize_url": "...", "certificate_url": "..."}

    certificate_ref VARCHAR(500), -- vault reference
    private_key_ref VARCHAR(500), -- vault reference
    issued_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    auto_renew      BOOLEAN NOT NULL DEFAULT true,
    renewal_config  JSONB NOT NULL DEFAULT '{}',
    -- renewal_config: {"days_before_expiry": 30, "preferred_challenge": "dns-01",
    --                   "notify_emails": ["ops@example.com"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certificates_property ON certificates(property_id);
CREATE INDEX idx_certificates_expiry ON certificates(expires_at);
```

---

## Purge Operations

```sql
CREATE TABLE purge_operations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    initiated_by    UUID REFERENCES users(id),
    purge_type      VARCHAR(50) NOT NULL
                    CHECK (purge_type IN ('url', 'surrogate_key', 'prefix', 'all')),
    purge_target    TEXT NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',

    -- Per-provider results stored as JSONB array
    provider_results JSONB NOT NULL DEFAULT '[]',
    -- provider_results example:
    -- [
    --   {"provider_slug": "cloudflare", "status": "completed", "propagation_ms": 320,
    --    "provider_purge_id": "cf-purge-abc123", "completed_at": "2026-05-20T10:30:01Z"},
    --   {"provider_slug": "fastly", "status": "completed", "propagation_ms": 142,
    --    "provider_purge_id": "fastly-purge-xyz", "completed_at": "2026-05-20T10:30:00.2Z"},
    --   {"provider_slug": "akamai", "status": "propagating",
    --    "provider_purge_id": "akamai-purge-999", "estimated_completion": "2026-05-20T10:35:00Z"}
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_purge_property ON purge_operations(property_id);
CREATE INDEX idx_purge_status ON purge_operations(status);
```

---

## Observability & Cost

```sql
CREATE TABLE edge_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    provider_slug   VARCHAR(50) NOT NULL,
    collected_at    TIMESTAMPTZ NOT NULL,
    granularity     VARCHAR(20) NOT NULL DEFAULT '1m'
                    CHECK (granularity IN ('1m', '5m', '1h', '1d')),

    -- Metrics as JSONB for flexible provider-specific metrics
    metrics         JSONB NOT NULL,
    -- metrics example:
    -- {"requests": 145230, "cache_hits": 130707, "cache_misses": 14523,
    --  "bytes_sent": 4893726481, "errors_4xx": 1204, "errors_5xx": 23,
    --  "latency_p50_ms": 12.4, "latency_p95_ms": 48.2, "latency_p99_ms": 124.6,
    --  "cache_hit_ratio": 0.90, "bandwidth_saved_pct": 0.85,
    --  "origin_offload_pct": 0.92}

    -- Geographic breakdown (optional, for detailed analytics)
    geo_breakdown   JSONB,
    -- geo_breakdown example:
    -- {"US": {"requests": 50000, "bytes": 1500000000, "p50_ms": 8.2},
    --  "DE": {"requests": 30000, "bytes": 900000000, "p50_ms": 15.4}}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_metrics_property_time ON edge_metrics(property_id, collected_at);
CREATE INDEX idx_metrics_provider_time ON edge_metrics(provider_slug, collected_at);

CREATE TABLE cost_tracking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider_slug   VARCHAR(50) NOT NULL,
    property_id     UUID REFERENCES properties(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,

    -- Cost details as JSONB for provider-specific billing models
    cost_details    JSONB NOT NULL,
    -- cost_details example:
    -- {"total_usd": 1234.56, "currency": "USD",
    --  "line_items": [
    --    {"type": "bandwidth", "bytes": 50000000000, "rate_per_gb": 0.02, "amount": 1000.00},
    --    {"type": "requests", "count": 100000000, "rate_per_10k": 0.01, "amount": 100.00},
    --    {"type": "edge_compute", "invocations": 5000000, "rate_per_million": 0.50, "amount": 2.50},
    --    {"type": "waf", "requests_inspected": 80000000, "amount": 132.06}
    --  ]}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cost_org_period ON cost_tracking(organization_id, period_start);
CREATE INDEX idx_cost_provider ON cost_tracking(provider_slug);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    -- changes example (JSON diff):
    -- {"before": {"max_age": 86400}, "after": {"max_age": 2592000}}
    request_context JSONB,
    -- request_context: {"ip": "203.0.113.42", "user_agent": "...",
    --                    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_time ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_changes ON audit_log USING GIN (changes);
```

---

## Example JSONB Queries

```sql
-- Find all cache rules that set stale-while-revalidate > 1 day
SELECT er.name, er.action->'directives'->>'stale_while_revalidate' AS swr
FROM edge_rules er
WHERE er.rule_type = 'cache'
  AND (er.action->'directives'->>'stale_while_revalidate')::int > 86400;

-- Find rules that match a specific path pattern
SELECT er.name, er.match_criteria
FROM edge_rules er
WHERE er.match_criteria @> '{"path": {"pattern": "/api/*"}}';

-- Find all properties with Cloudflare as a traffic target
SELECT p.name, tp.targets
FROM properties p
JOIN traffic_policies tp ON tp.property_id = p.id
WHERE tp.targets @> '[{"provider_slug": "cloudflare"}]';

-- Find rules drifted at any provider
SELECT er.name, prc.provider_slug, prc.drift_diff
FROM edge_rules er
JOIN provider_rule_configs prc ON prc.edge_rule_id = er.id
WHERE prc.sync_status = 'drifted';

-- Cost breakdown by provider for an organization
SELECT provider_slug,
       SUM((cost_details->>'total_usd')::numeric) AS total_cost
FROM cost_tracking
WHERE organization_id = $1
  AND period_start >= '2026-01-01'
GROUP BY provider_slug
ORDER BY total_cost DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Users | 3 | Multi-tenant RBAC with JSONB permissions |
| Provider Connections | 1 | Auth config and capabilities in JSONB |
| Properties & Domains | 3 | Protocol config in JSONB |
| Configuration Versioning | 1 | Validation results in JSONB |
| Edge Rules | 2 | Abstract rules + provider-native configs |
| Traffic Steering | 1 | Targets and geo overrides in JSONB |
| Certificates | 1 | ACME state in JSONB |
| Purge Operations | 1 | Per-provider results in JSONB |
| Observability | 2 | Flexible metrics and cost details in JSONB |
| Audit | 1 | Change diffs in JSONB |
| **Total** | **16** | Moderate table count; complexity in JSONB structures |

---

## Key Design Decisions

1. **Dual-representation for edge rules.** Each rule has an abstract `match_criteria` + `action` (the platform's portable format) and a per-provider `native_config` (the exact Cloudflare expression, Fastly VCL, or Akamai behavior). This avoids lowest-common-denominator abstraction while maintaining cross-provider visibility.

2. **Provider-specific configuration in JSONB, not in provider-specific tables.** Adding support for a new CDN provider (e.g., Azure CDN) requires no schema migration — only new JSONB structures in `provider_rule_configs.native_config` and `provider_connections.auth_config`.

3. **GIN indexes on all JSONB columns used for querying.** The `match_criteria`, `action`, `targets`, `geo_overrides`, `native_config`, and `changes` columns all have GIN indexes to support JSONB containment queries (`@>`).

4. **Purge results as a JSONB array** rather than a separate table. Since purge operations are short-lived and the number of providers per property is small (typically 2-5), storing per-provider results as a JSONB array on the purge operation avoids an extra table and JOIN.

5. **Metrics and cost details in JSONB** because billing models vary wildly across providers (per-GB, per-request, per-compute-invocation, bundled WAF). A JSONB `line_items` array accommodates any provider's billing structure without schema changes.

6. **AI analysis metadata stored alongside rules.** The `ai_analysis` JSONB column on `edge_rules` stores ML pipeline outputs (recommended TTLs, cache hit predictions, risk scores) directly on the rule, making it easy to display recommendations in the UI without a separate lookup.

7. **JSON Schema validation enforced at the application layer.** Every JSONB column has a documented expected structure (shown in comments above), and the application validates against JSON Schema (Draft 2020-12) before writing. This compensates for the lack of database-level structural constraints on JSONB.
