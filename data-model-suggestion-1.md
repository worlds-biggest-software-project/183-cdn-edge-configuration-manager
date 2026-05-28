# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: CDN & Edge Configuration Manager · Created: 2026-05-20

## Philosophy

This model follows traditional third-normal-form (3NF) relational design, giving every concept in the CDN configuration domain its own table with explicit foreign key relationships. Each CDN provider, domain, backend origin, cache rule, security policy, and traffic-steering rule is a first-class entity with dedicated columns and constraints. The schema enforces referential integrity at the database level, making it impossible to create orphaned rules or reference non-existent providers.

This approach mirrors how Akamai's Property Manager API structures its data: properties own versioned rule trees, each rule contains behaviors and criteria, and every object has a well-defined type and lifecycle. Cloudflare's zone/ruleset/rule hierarchy and Fastly's service/domain/backend/condition model similarly decompose configuration into discrete, typed entities. By normalizing fully, the schema supports complex cross-entity queries (e.g., "which cache rules across all providers reference this origin?") without JSONB containment operators.

The trade-off is table count and migration overhead. Adding a new CDN provider capability (e.g., a new Fastly condition type) requires a schema migration rather than a JSONB field update. However, for a platform where configuration correctness is safety-critical (misconfigured cache rules can expose private data), the discipline of explicit column definitions and CHECK constraints is a significant advantage.

**Best for:** Teams that prioritize data integrity, complex cross-entity reporting, and regulatory audit requirements over rapid schema evolution.

**Trade-offs:**
- Pro: Strong referential integrity; complex queries are straightforward SQL JOINs
- Pro: Database-level enforcement of valid configurations via CHECK constraints and foreign keys
- Pro: Standard ORM tooling works naturally; no JSONB-specific query patterns needed
- Con: High table count (~45-55 tables); schema migrations required for new provider features
- Con: Multi-provider abstraction requires junction tables and polymorphic patterns
- Con: Less flexible for provider-specific fields that vary widely across CDN vendors

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 9111 (HTTP Caching) | Cache directive columns map directly to RFC 9111 response directives (max-age, s-maxage, must-revalidate, no-cache, no-store, stale-while-revalidate, stale-if-error) |
| RFC 9213 (CDN-Cache-Control) | Dedicated `targeted_cache_directives` table models per-CDN-tier cache control as Dictionary Structured Fields |
| RFC 9114 / RFC 9000 (HTTP/3 / QUIC) | Protocol configuration table includes QUIC-specific fields (connection_migration, zero_rtt, initial_cwnd) |
| RFC 8555 (ACME) | Certificate table models ACME order lifecycle (pending, ready, processing, valid, invalid) |
| ISO 3166-1 (Country codes) | Geography reference table uses ISO 3166-1 alpha-2 codes for geo-routing and geo-blocking rules |
| OpenTelemetry Semantic Conventions | Observability metric columns follow OTEL HTTP semantic conventions (http.request.method, http.response.status_code) |
| W3C ESI 1.0 | ESI template table models include/remove/try directives per W3C Edge Side Includes spec |
| OWASP Top 10 | WAF rule categories mapped to OWASP classification IDs |

---

## Core Platform Tables

### Organizations & Tenancy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free'
                    CHECK (plan_tier IN ('free', 'pro', 'business', 'enterprise')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'admin', 'editor', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_members(organization_id);
CREATE INDEX idx_org_members_user ON organization_members(user_id);
```

### CDN Provider Integration

```sql
CREATE TABLE cdn_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            VARCHAR(50) NOT NULL UNIQUE,
    -- e.g., 'cloudflare', 'fastly', 'akamai', 'aws_cloudfront', 'gcore', 'bunnycdn'
    display_name    VARCHAR(100) NOT NULL,
    api_base_url    VARCHAR(500) NOT NULL,
    auth_method     VARCHAR(50) NOT NULL
                    CHECK (auth_method IN ('bearer_token', 'api_key', 'oauth2', 'edgegrid', 'sigv4')),
    supports_http3  BOOLEAN NOT NULL DEFAULT false,
    supports_quic   BOOLEAN NOT NULL DEFAULT false,
    supports_esi    BOOLEAN NOT NULL DEFAULT false,
    supports_edge_compute BOOLEAN NOT NULL DEFAULT false,
    purge_propagation_ms INTEGER, -- typical purge propagation time
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE provider_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    credential_name VARCHAR(255) NOT NULL,
    -- Encrypted credential storage; actual secrets in vault
    api_token_ref   VARCHAR(500), -- reference to secret in vault (e.g., vault://cdn/cloudflare/token)
    -- Akamai EdgeGrid specific
    client_token    VARCHAR(255),
    access_token    VARCHAR(255),
    client_secret_ref VARCHAR(500),
    edgegrid_host   VARCHAR(500),
    -- AWS SigV4 specific
    aws_access_key_ref VARCHAR(500),
    aws_region      VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_validated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, cdn_provider_id, credential_name)
);

CREATE INDEX idx_provider_creds_org ON provider_credentials(organization_id);
```

### Properties & Domains

```sql
CREATE TABLE properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    environment     VARCHAR(50) NOT NULL DEFAULT 'production'
                    CHECK (environment IN ('development', 'staging', 'production')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, slug)
);

CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    hostname        VARCHAR(500) NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    tls_enabled     BOOLEAN NOT NULL DEFAULT true,
    min_tls_version VARCHAR(10) DEFAULT '1.2'
                    CHECK (min_tls_version IN ('1.0', '1.1', '1.2', '1.3')),
    http3_enabled   BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_domains_hostname ON domains(hostname);
CREATE INDEX idx_domains_property ON domains(property_id);

CREATE TABLE origins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    hostname        VARCHAR(500) NOT NULL,
    port            INTEGER NOT NULL DEFAULT 443,
    protocol        VARCHAR(10) NOT NULL DEFAULT 'https'
                    CHECK (protocol IN ('http', 'https')),
    weight          INTEGER NOT NULL DEFAULT 100, -- for load balancing
    is_backup       BOOLEAN NOT NULL DEFAULT false,
    health_check_path VARCHAR(500) DEFAULT '/health',
    health_check_interval_sec INTEGER DEFAULT 30,
    connect_timeout_ms INTEGER DEFAULT 5000,
    read_timeout_ms INTEGER DEFAULT 30000,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_origins_property ON origins(property_id);
```

### Configuration Versioning

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
    deactivated_at  TIMESTAMPTZ,
    parent_version_id UUID REFERENCES config_versions(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(property_id, version_number)
);

CREATE INDEX idx_config_versions_property ON config_versions(property_id);
CREATE INDEX idx_config_versions_status ON config_versions(status);
```

### Cache Rules (RFC 9111 / RFC 9213 aligned)

```sql
CREATE TABLE cache_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_version_id UUID NOT NULL REFERENCES config_versions(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0, -- lower = higher priority
    is_enabled      BOOLEAN NOT NULL DEFAULT true,

    -- Match criteria
    match_path_pattern    VARCHAR(1000), -- glob or regex
    match_path_type       VARCHAR(10) DEFAULT 'glob'
                          CHECK (match_path_type IN ('glob', 'regex', 'exact')),
    match_method          VARCHAR(10)[], -- e.g., {'GET', 'HEAD'}
    match_content_type    VARCHAR(255)[], -- e.g., {'image/*', 'text/css'}
    match_query_string    VARCHAR(500),
    match_header_name     VARCHAR(255),
    match_header_value    VARCHAR(500),

    -- RFC 9111 Cache-Control directives
    cache_action          VARCHAR(50) NOT NULL DEFAULT 'cache'
                          CHECK (cache_action IN ('cache', 'bypass', 'revalidate', 'custom')),
    max_age_seconds       INTEGER, -- Cache-Control: max-age
    s_maxage_seconds      INTEGER, -- Cache-Control: s-maxage
    stale_while_revalidate_seconds INTEGER, -- stale-while-revalidate (RFC 5861)
    stale_if_error_seconds INTEGER, -- stale-if-error (RFC 5861)
    must_revalidate       BOOLEAN DEFAULT false,
    no_cache              BOOLEAN DEFAULT false,
    no_store              BOOLEAN DEFAULT false,
    is_private            BOOLEAN DEFAULT false,
    no_transform          BOOLEAN DEFAULT false,

    -- Surrogate/CDN-specific (RFC 9213)
    cdn_cache_control_max_age INTEGER, -- CDN-Cache-Control: max-age
    surrogate_key         VARCHAR(500), -- for tag-based purging

    -- Edge behavior
    serve_stale_on_error  BOOLEAN DEFAULT true,
    respect_origin_headers BOOLEAN DEFAULT true,
    strip_query_string    BOOLEAN DEFAULT false,

    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cache_rules_version ON cache_rules(config_version_id);
CREATE INDEX idx_cache_rules_priority ON cache_rules(config_version_id, priority);
```

### Security & WAF Rules

```sql
CREATE TABLE waf_rulesets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_version_id UUID NOT NULL REFERENCES config_versions(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    mode            VARCHAR(50) NOT NULL DEFAULT 'block'
                    CHECK (mode IN ('block', 'simulate', 'challenge', 'disabled')),
    owasp_category  VARCHAR(50), -- e.g., 'A01:2021-Broken Access Control'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE security_headers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_version_id UUID NOT NULL REFERENCES config_versions(id) ON DELETE CASCADE,
    header_name     VARCHAR(255) NOT NULL,
    header_value    TEXT NOT NULL,
    action          VARCHAR(20) NOT NULL DEFAULT 'set'
                    CHECK (action IN ('set', 'append', 'remove')),
    apply_to        VARCHAR(20) NOT NULL DEFAULT 'response'
                    CHECK (apply_to IN ('request', 'response')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_waf_rulesets_version ON waf_rulesets(config_version_id);
CREATE INDEX idx_security_headers_version ON security_headers(config_version_id);
```

### TLS / Certificate Management (ACME-aligned)

```sql
CREATE TABLE certificates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    domain_names    VARCHAR(500)[] NOT NULL, -- SAN list
    issuer          VARCHAR(255), -- e.g., 'letsencrypt', 'zerossl', 'digicert'
    acme_order_status VARCHAR(50)
                    CHECK (acme_order_status IN ('pending', 'ready', 'processing', 'valid', 'invalid')),
    certificate_pem_ref VARCHAR(500), -- vault reference
    private_key_ref VARCHAR(500), -- vault reference
    issued_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    auto_renew      BOOLEAN NOT NULL DEFAULT true,
    renewal_days_before_expiry INTEGER DEFAULT 30,
    last_renewal_attempt TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certificates_property ON certificates(property_id);
CREATE INDEX idx_certificates_expiry ON certificates(expires_at);
```

### Traffic Steering & Multi-CDN Routing

```sql
CREATE TABLE traffic_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    config_version_id UUID NOT NULL REFERENCES config_versions(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    strategy        VARCHAR(50) NOT NULL DEFAULT 'weighted'
                    CHECK (strategy IN ('weighted', 'geolocation', 'latency', 'failover', 'cost', 'ml_optimized')),
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE traffic_policy_targets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    traffic_policy_id UUID NOT NULL REFERENCES traffic_policies(id) ON DELETE CASCADE,
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    credential_id   UUID NOT NULL REFERENCES provider_credentials(id),
    weight          INTEGER NOT NULL DEFAULT 100,
    priority        INTEGER NOT NULL DEFAULT 0, -- for failover ordering
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    max_cost_per_gb_usd NUMERIC(10, 6),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE geo_routing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    traffic_policy_id UUID NOT NULL REFERENCES traffic_policies(id) ON DELETE CASCADE,
    country_code    CHAR(2) NOT NULL, -- ISO 3166-1 alpha-2
    region_code     VARCHAR(10), -- ISO 3166-2 subdivision
    continent_code  CHAR(2), -- AF, AN, AS, EU, NA, OC, SA
    target_id       UUID NOT NULL REFERENCES traffic_policy_targets(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_traffic_policies_property ON traffic_policies(property_id);
CREATE INDEX idx_traffic_targets_policy ON traffic_policy_targets(traffic_policy_id);
CREATE INDEX idx_geo_routing_policy ON geo_routing_rules(traffic_policy_id);
CREATE INDEX idx_geo_routing_country ON geo_routing_rules(country_code);
```

### Purge Operations

```sql
CREATE TABLE purge_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    initiated_by    UUID REFERENCES users(id),
    purge_type      VARCHAR(50) NOT NULL
                    CHECK (purge_type IN ('url', 'surrogate_key', 'prefix', 'all')),
    purge_target    TEXT NOT NULL, -- URL, surrogate key, or prefix
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'propagating', 'completed', 'failed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE TABLE purge_request_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purge_request_id UUID NOT NULL REFERENCES purge_requests(id) ON DELETE CASCADE,
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'submitted', 'completed', 'failed')),
    provider_purge_id VARCHAR(255), -- provider's own purge ID
    propagation_ms  INTEGER,
    error_message   TEXT,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_purge_requests_property ON purge_requests(property_id);
CREATE INDEX idx_purge_results_request ON purge_request_results(purge_request_id);
```

### Provider Sync State

```sql
CREATE TABLE provider_sync_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_version_id UUID NOT NULL REFERENCES config_versions(id),
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    credential_id   UUID NOT NULL REFERENCES provider_credentials(id),
    sync_status     VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (sync_status IN ('pending', 'syncing', 'synced', 'drifted', 'failed')),
    provider_resource_id VARCHAR(500), -- e.g., Cloudflare zone ID, Fastly service ID
    provider_version_id VARCHAR(255), -- e.g., Akamai property version, Fastly service version
    last_synced_at  TIMESTAMPTZ,
    last_drift_check_at TIMESTAMPTZ,
    drift_details   TEXT,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(config_version_id, cdn_provider_id)
);

CREATE INDEX idx_sync_state_version ON provider_sync_state(config_version_id);
CREATE INDEX idx_sync_state_status ON provider_sync_state(sync_status);
```

### Observability & Metrics

```sql
CREATE TABLE edge_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    collected_at    TIMESTAMPTZ NOT NULL,
    -- OTEL-aligned metric names
    requests_total  BIGINT NOT NULL DEFAULT 0,
    cache_hits      BIGINT NOT NULL DEFAULT 0,
    cache_misses    BIGINT NOT NULL DEFAULT 0,
    bytes_sent      BIGINT NOT NULL DEFAULT 0,
    error_count_4xx BIGINT NOT NULL DEFAULT 0,
    error_count_5xx BIGINT NOT NULL DEFAULT 0,
    p50_latency_ms  NUMERIC(10, 2),
    p95_latency_ms  NUMERIC(10, 2),
    p99_latency_ms  NUMERIC(10, 2),
    country_code    CHAR(2), -- ISO 3166-1
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edge_metrics_property_time ON edge_metrics(property_id, collected_at);
CREATE INDEX idx_edge_metrics_provider_time ON edge_metrics(cdn_provider_id, collected_at);

-- Cost tracking
CREATE TABLE cost_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    property_id     UUID REFERENCES properties(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    bytes_transferred BIGINT NOT NULL DEFAULT 0,
    requests_total  BIGINT NOT NULL DEFAULT 0,
    cost_usd        NUMERIC(12, 4) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cost_records_org_period ON cost_records(organization_id, period_start);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    -- e.g., 'config_version.activate', 'cache_rule.create', 'purge.initiate'
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID,
    details         TEXT,
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org_time ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Users | 3 | Multi-tenant RBAC |
| CDN Providers & Credentials | 2 | Provider registry + encrypted credential refs |
| Properties & Domains | 3 | Property/domain/origin hierarchy |
| Configuration Versioning | 1 | Immutable version management |
| Cache Rules | 1 | RFC 9111/9213 aligned directives |
| Security & WAF | 2 | WAF rulesets + security headers |
| TLS / Certificates | 1 | ACME lifecycle |
| Traffic Steering | 3 | Multi-CDN routing with geo rules |
| Purge Operations | 2 | Cross-provider purge tracking |
| Provider Sync | 1 | Drift detection per provider |
| Observability | 2 | Metrics + cost records |
| Audit | 1 | Immutable audit log |
| **Total** | **22** | Core schema; extensible with additional rule types |

---

## Key Design Decisions

1. **Explicit RFC 9111 columns on cache_rules** rather than a generic key-value store. Each Cache-Control directive (max-age, s-maxage, stale-while-revalidate, etc.) is a typed column with appropriate constraints. This makes validation trivial and queries like "find all rules with stale-while-revalidate > 86400" a simple WHERE clause.

2. **Configuration versioning follows Akamai's immutable activation model.** Once a config_version is activated, it cannot be modified — changes require creating a new version. The parent_version_id field creates a linked list of version history.

3. **Provider credentials reference a vault** rather than storing secrets directly. The `_ref` columns contain vault URIs (e.g., `vault://cdn/cloudflare/org-123/token`) rather than plaintext secrets.

4. **Traffic steering uses a policy/target/geo-rule three-table pattern** to support weighted, geolocation, latency, failover, and ML-optimized strategies without schema changes.

5. **Purge operations are tracked per-provider** because propagation times vary (150ms for Fastly surrogate-key purge vs. minutes for Akamai). The purge_request_results table captures per-provider status independently.

6. **Edge metrics use ISO 3166-1 country codes** for geographic attribution, aligning with CDN provider analytics APIs that report by country.

7. **Audit log is append-only** with no UPDATE or DELETE operations expected. The action field uses a `resource_type.verb` convention for structured querying.
