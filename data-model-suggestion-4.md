# Data Model Suggestion 4: Graph-Relational

> Project: CDN & Edge Configuration Manager · Created: 2026-05-20

## Philosophy

This model uses a property-graph layer on top of relational tables to model the inherently graph-shaped relationships in CDN configuration: properties connect to multiple providers, rules depend on other rules (priority chains, override hierarchies), providers serve geographic regions through points of presence, traffic flows from domains through edge nodes to origins, and configuration changes propagate across a dependency graph of resources. Instead of expressing these relationships through junction tables and recursive CTEs, the graph layer makes them first-class traversable edges.

The design uses PostgreSQL's `ltree` extension for hierarchical rule trees (mirroring Akamai's rule tree structure of up to five levels deep) and a lightweight `graph_edges` table for cross-entity relationships (traffic flow paths, dependency chains, conflict detection). The relational tables handle CRUD operations and transactional integrity, while the graph edges enable queries like "what is the blast radius if this origin goes down?" (traverse all properties and providers that route to it) or "which rules conflict with this new cache rule?" (find overlapping path patterns in the rule tree).

This approach is inspired by how large-scale CDN platforms think about their infrastructure: as a network graph of PoPs, origins, and traffic flows. IO River's multi-CDN orchestration, for example, fundamentally reasons about traffic as a graph of paths between end users, edge nodes, and origins. Akamai's rule tree is explicitly a tree (a restricted graph). The graph-relational model makes these implicit graphs explicit and queryable.

**Best for:** Platforms emphasizing multi-CDN traffic flow analysis, blast-radius assessment, conflict detection, dependency tracking, and complex relationship queries between configuration entities.

**Trade-offs:**
- Pro: Natural representation of CDN topology (PoPs, origins, traffic flows)
- Pro: Powerful graph traversal queries for blast-radius and dependency analysis
- Pro: Hierarchical rule trees modeled natively with ltree (efficient ancestor/descendant queries)
- Pro: Conflict detection via graph overlap analysis
- Con: Requires PostgreSQL ltree extension (not portable to all databases)
- Con: Graph edge maintenance adds write overhead
- Con: Developers need to understand graph query patterns (ltree operators, recursive CTEs)
- Con: More complex than a pure relational model for simple CRUD operations

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 9111 (HTTP Caching) | Cache directives stored on rule nodes; the rule tree determines directive inheritance and override order |
| RFC 9213 (CDN-Cache-Control) | CDN-targeted directives modeled as rule node attributes with provider-specific edge labels |
| RFC 9114 / RFC 9000 (HTTP/3 / QUIC) | Protocol capabilities modeled as PoP node attributes; QUIC support as a graph property |
| W3C ESI 1.0 | ESI template composition modeled as a subgraph of fragment dependencies |
| ISO 3166-1 | Geographic PoP locations use ISO country codes; geo-routing traverses country-to-PoP edges |
| Akamai Rule Tree Model | Rule tree hierarchy directly modeled using ltree paths (up to 5 levels deep) |
| OpenTelemetry | Trace context propagated through graph edges for distributed tracing correlation |

---

## Core Relational Tables

### Organizations & Users

```sql
CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, user_id)
);
```

### CDN Providers & Points of Presence

```sql
CREATE TABLE cdn_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            VARCHAR(50) NOT NULL UNIQUE,
    display_name    VARCHAR(100) NOT NULL,
    api_base_url    VARCHAR(500) NOT NULL,
    auth_method     VARCHAR(50) NOT NULL,
    capabilities    JSONB NOT NULL DEFAULT '{}',
    -- capabilities: {"http3": true, "esi": false, "edge_compute": true,
    --                "purge_methods": ["url", "surrogate_key", "all"],
    --                "purge_propagation_ms": 150}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Points of Presence: the physical/logical edge locations for each provider
CREATE TABLE points_of_presence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    pop_code        VARCHAR(20) NOT NULL, -- e.g., 'IAD', 'FRA', 'NRT', 'SIN'
    city            VARCHAR(255),
    country_code    CHAR(2) NOT NULL, -- ISO 3166-1 alpha-2
    continent_code  CHAR(2) NOT NULL, -- AF, AN, AS, EU, NA, OC, SA
    latitude        NUMERIC(9, 6),
    longitude       NUMERIC(9, 6),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(cdn_provider_id, pop_code)
);

CREATE INDEX idx_pops_provider ON points_of_presence(cdn_provider_id);
CREATE INDEX idx_pops_country ON points_of_presence(country_code);
CREATE INDEX idx_pops_continent ON points_of_presence(continent_code);

CREATE TABLE provider_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    credential_name VARCHAR(255) NOT NULL,
    auth_config     JSONB NOT NULL, -- provider-specific auth details (vault refs)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_validated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, cdn_provider_id, credential_name)
);
```

### Properties, Domains & Origins

```sql
CREATE TABLE properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, slug)
);

CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    hostname        VARCHAR(500) NOT NULL UNIQUE,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    tls_min_version VARCHAR(10) DEFAULT '1.2',
    http3_enabled   BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE origins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    hostname        VARCHAR(500) NOT NULL,
    port            INTEGER NOT NULL DEFAULT 443,
    protocol        VARCHAR(10) NOT NULL DEFAULT 'https',
    weight          INTEGER NOT NULL DEFAULT 100,
    is_backup       BOOLEAN NOT NULL DEFAULT false,
    health_check_path VARCHAR(500) DEFAULT '/health',
    connect_timeout_ms INTEGER DEFAULT 5000,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_domains_property ON domains(property_id);
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(property_id, version_number)
);

CREATE INDEX idx_config_versions_property ON config_versions(property_id);
```

---

## Rule Tree (ltree-based hierarchical rules)

```sql
-- Rules organized as a tree, mirroring Akamai's rule tree model.
-- The ltree path encodes the hierarchy: 'root' -> 'root.static' -> 'root.static.images'
-- Each level can override or extend parent directives.
CREATE TABLE rule_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_version_id UUID NOT NULL REFERENCES config_versions(id) ON DELETE CASCADE,
    tree_path       ltree NOT NULL,
    -- e.g., 'root', 'root.static_assets', 'root.static_assets.images',
    --        'root.api', 'root.api.v2', 'root.api.v2.authenticated'
    name            VARCHAR(255) NOT NULL,
    rule_type       VARCHAR(50) NOT NULL
                    CHECK (rule_type IN ('default', 'cache', 'header', 'redirect',
                                         'rewrite', 'rate_limit', 'waf', 'geo_block', 'esi')),
    priority        INTEGER NOT NULL DEFAULT 0,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,

    -- Match criteria (when does this rule apply?)
    match_path_pattern VARCHAR(1000),
    match_path_type    VARCHAR(10) DEFAULT 'glob',
    match_methods      VARCHAR(10)[], -- {'GET', 'HEAD'}
    match_content_types VARCHAR(255)[],
    match_conditions   JSONB, -- complex conditions beyond simple path matching

    -- Cache directives (RFC 9111)
    cache_action    VARCHAR(50),
    max_age_seconds INTEGER,
    s_maxage_seconds INTEGER,
    stale_while_revalidate_seconds INTEGER,
    stale_if_error_seconds INTEGER,
    cdn_cache_control_max_age INTEGER,
    surrogate_key   VARCHAR(500),
    must_revalidate BOOLEAN DEFAULT false,
    no_cache        BOOLEAN DEFAULT false,
    no_store        BOOLEAN DEFAULT false,

    -- Header modifications
    headers_to_set  JSONB, -- [{"name": "X-Frame-Options", "value": "DENY", "target": "response"}]
    headers_to_remove VARCHAR(255)[],

    -- WAF/security
    waf_mode        VARCHAR(50),
    waf_owasp_categories VARCHAR(50)[],

    -- Provider-native overrides (per-provider if the abstract rule doesn't translate cleanly)
    provider_overrides JSONB,
    -- provider_overrides example:
    -- {"cloudflare": {"expression": "custom cf expression"},
    --  "fastly": {"vcl_snippet": "custom VCL"}}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ltree indexes for hierarchical queries
CREATE INDEX idx_rule_nodes_path ON rule_nodes USING GIST (tree_path);
CREATE INDEX idx_rule_nodes_version ON rule_nodes(config_version_id);
CREATE INDEX idx_rule_nodes_type ON rule_nodes(rule_type);
```

### Rule Tree Query Examples

```sql
-- Find all descendant rules under 'root.static_assets' (the entire subtree)
SELECT * FROM rule_nodes
WHERE config_version_id = $1
  AND tree_path <@ 'root.static_assets'
ORDER BY tree_path;

-- Find all ancestor rules for a specific deep rule (for directive inheritance)
SELECT * FROM rule_nodes
WHERE config_version_id = $1
  AND tree_path @> 'root.api.v2.authenticated'
ORDER BY nlevel(tree_path);

-- Find sibling rules at the same level (for conflict detection)
SELECT * FROM rule_nodes
WHERE config_version_id = $1
  AND tree_path ~ 'root.api.*{1}'
ORDER BY priority;

-- Find all cache rules at any depth that set max_age > 30 days
SELECT name, tree_path, max_age_seconds
FROM rule_nodes
WHERE config_version_id = $1
  AND rule_type = 'cache'
  AND max_age_seconds > 2592000;

-- Compute effective cache directives for a path by walking ancestors
-- (child rules override parent rules)
WITH RECURSIVE effective_rules AS (
    SELECT rn.*, nlevel(rn.tree_path) AS depth
    FROM rule_nodes rn
    WHERE rn.config_version_id = $1
      AND rn.tree_path @> 'root.api.v2.authenticated'
      AND rn.is_enabled = true
    ORDER BY nlevel(rn.tree_path)
)
SELECT * FROM effective_rules ORDER BY depth;
```

---

## Graph Edges (Cross-Entity Relationships)

```sql
-- Generic graph edge table for relationships between any entities.
-- Enables traversal queries: blast radius, dependency analysis, conflict detection.
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type     VARCHAR(50) NOT NULL,
    source_id       UUID NOT NULL,
    edge_type       VARCHAR(100) NOT NULL,
    target_type     VARCHAR(50) NOT NULL,
    target_id       UUID NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Edge-specific metadata. Examples:
    -- For 'routes_traffic_to': {"weight": 60, "strategy": "weighted"}
    -- For 'serves_region': {"latency_ms": 12, "capacity_gbps": 100}
    -- For 'depends_on': {"dependency_type": "hard", "reason": "origin fallback"}
    -- For 'conflicts_with': {"overlap_type": "path_pattern", "severity": "warning"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(source_type, source_id, edge_type, target_type, target_id)
);

CREATE INDEX idx_edges_source ON graph_edges(source_type, source_id);
CREATE INDEX idx_edges_target ON graph_edges(target_type, target_id);
CREATE INDEX idx_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_edges_properties ON graph_edges USING GIN (properties);
```

### Edge Type Catalog

| Edge Type | Source | Target | Meaning |
|-----------|--------|--------|---------|
| `routes_traffic_to` | Property | CDN Provider | Property uses this provider for traffic |
| `serves_region` | PoP | Country (ISO 3166-1) | This PoP serves this geographic region |
| `has_origin` | Property | Origin | Property routes cache misses to this origin |
| `depends_on` | Origin | Origin | Backup/failover dependency between origins |
| `conflicts_with` | Rule Node | Rule Node | Two rules have overlapping match criteria |
| `overrides` | Rule Node | Rule Node | Child rule overrides parent rule directive |
| `certificate_covers` | Certificate | Domain | TLS cert covers this domain (SAN) |
| `synced_to` | Config Version | CDN Provider | Version has been synced to this provider |
| `purge_targets` | Purge Request | CDN Provider | Purge sent to this provider |

### Graph Traversal Query Examples

```sql
-- Blast radius: if origin-1 goes down, which properties are affected?
WITH RECURSIVE blast AS (
    -- Start from the failing origin
    SELECT ge.target_type, ge.target_id, ge.edge_type, 1 AS depth
    FROM graph_edges ge
    WHERE ge.source_type = 'Origin'
      AND ge.source_id = $origin_id
      AND ge.edge_type IN ('has_origin', 'depends_on')

    UNION ALL

    -- Traverse outward
    SELECT ge2.target_type, ge2.target_id, ge2.edge_type, b.depth + 1
    FROM blast b
    JOIN graph_edges ge2 ON ge2.source_id = b.target_id
    WHERE b.depth < 5 -- prevent infinite loops
)
SELECT DISTINCT target_type, target_id FROM blast;

-- Which providers serve Japan? (traverse PoP -> Country edges)
SELECT cp.display_name, pop.pop_code, pop.city
FROM graph_edges ge
JOIN points_of_presence pop ON pop.id = ge.source_id
JOIN cdn_providers cp ON cp.id = pop.cdn_provider_id
WHERE ge.edge_type = 'serves_region'
  AND ge.target_type = 'Country'
  AND ge.properties->>'country_code' = 'JP';

-- Find conflicting rules within a config version
SELECT rn1.name AS rule_a, rn1.tree_path AS path_a,
       rn2.name AS rule_b, rn2.tree_path AS path_b,
       ge.properties->>'overlap_type' AS overlap,
       ge.properties->>'severity' AS severity
FROM graph_edges ge
JOIN rule_nodes rn1 ON rn1.id = ge.source_id
JOIN rule_nodes rn2 ON rn2.id = ge.target_id
WHERE ge.edge_type = 'conflicts_with'
  AND rn1.config_version_id = $1;

-- Full traffic path: domain -> property -> providers -> PoPs
SELECT d.hostname, p.name AS property,
       cp.display_name AS provider, pop.pop_code, pop.city
FROM domains d
JOIN properties p ON p.id = d.property_id
JOIN graph_edges ge1 ON ge1.source_type = 'Property' AND ge1.source_id = p.id
    AND ge1.edge_type = 'routes_traffic_to'
JOIN cdn_providers cp ON cp.id = ge1.target_id
JOIN points_of_presence pop ON pop.cdn_provider_id = cp.id
WHERE d.hostname = 'cdn.example.com';
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Traffic targets are also represented as graph edges (routes_traffic_to)
-- but this table stores the operational parameters
CREATE TABLE traffic_targets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    traffic_policy_id UUID NOT NULL REFERENCES traffic_policies(id) ON DELETE CASCADE,
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    credential_id   UUID NOT NULL REFERENCES provider_credentials(id),
    weight          INTEGER NOT NULL DEFAULT 100,
    priority        INTEGER NOT NULL DEFAULT 0,
    max_cost_per_gb_usd NUMERIC(10, 6),
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE geo_routing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    traffic_policy_id UUID NOT NULL REFERENCES traffic_policies(id) ON DELETE CASCADE,
    country_code    CHAR(2) NOT NULL, -- ISO 3166-1
    continent_code  CHAR(2),
    target_id       UUID NOT NULL REFERENCES traffic_targets(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_traffic_policies_property ON traffic_policies(property_id);
CREATE INDEX idx_traffic_targets_policy ON traffic_targets(traffic_policy_id);
CREATE INDEX idx_geo_routing_country ON geo_routing_rules(country_code);
```

---

## Certificates, Purges, Sync State

```sql
CREATE TABLE certificates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    domain_names    VARCHAR(500)[] NOT NULL,
    issuer          VARCHAR(255),
    acme_order_status VARCHAR(50),
    certificate_ref VARCHAR(500),
    private_key_ref VARCHAR(500),
    issued_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    auto_renew      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Certificate-to-domain edges tracked in graph_edges (certificate_covers)

CREATE TABLE purge_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    initiated_by    UUID REFERENCES users(id),
    purge_type      VARCHAR(50) NOT NULL,
    purge_target    TEXT NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE TABLE purge_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purge_request_id UUID NOT NULL REFERENCES purge_requests(id) ON DELETE CASCADE,
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    propagation_ms  INTEGER,
    provider_purge_id VARCHAR(255),
    error_message   TEXT,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE provider_sync_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_version_id UUID NOT NULL REFERENCES config_versions(id),
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    sync_status     VARCHAR(50) NOT NULL DEFAULT 'pending',
    provider_resource_id VARCHAR(500),
    provider_version_id VARCHAR(255),
    last_synced_at  TIMESTAMPTZ,
    drift_details   TEXT,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(config_version_id, cdn_provider_id)
);

CREATE INDEX idx_certificates_property ON certificates(property_id);
CREATE INDEX idx_purge_requests_property ON purge_requests(property_id);
CREATE INDEX idx_purge_results_request ON purge_results(purge_request_id);
CREATE INDEX idx_sync_state_version ON provider_sync_state(config_version_id);
```

---

## Observability & Cost

```sql
CREATE TABLE edge_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    cdn_provider_id UUID NOT NULL REFERENCES cdn_providers(id),
    pop_id          UUID REFERENCES points_of_presence(id),
    -- Per-PoP metrics enable geographic performance analysis
    collected_at    TIMESTAMPTZ NOT NULL,
    requests_total  BIGINT NOT NULL DEFAULT 0,
    cache_hits      BIGINT NOT NULL DEFAULT 0,
    cache_misses    BIGINT NOT NULL DEFAULT 0,
    bytes_sent      BIGINT NOT NULL DEFAULT 0,
    error_count_4xx BIGINT NOT NULL DEFAULT 0,
    error_count_5xx BIGINT NOT NULL DEFAULT 0,
    p50_latency_ms  NUMERIC(10, 2),
    p95_latency_ms  NUMERIC(10, 2),
    p99_latency_ms  NUMERIC(10, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_metrics_property_time ON edge_metrics(property_id, collected_at);
CREATE INDEX idx_metrics_pop_time ON edge_metrics(pop_id, collected_at);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cost_org_period ON cost_records(organization_id, period_start);
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
    details         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_time ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Users | 3 | Standard multi-tenant RBAC |
| CDN Providers & PoPs | 3 | Provider registry + PoP geography + credentials |
| Properties & Domains | 3 | Property/domain/origin hierarchy |
| Configuration Versioning | 1 | Immutable versions |
| Rule Tree | 1 | ltree-based hierarchical rules (replaces multiple rule tables) |
| Graph Edges | 1 | Generic cross-entity relationship graph |
| Traffic Steering | 3 | Policies, targets, geo rules |
| Certificates | 1 | ACME lifecycle |
| Purge Operations | 2 | Request + per-provider results |
| Provider Sync | 1 | Drift detection |
| Observability | 2 | Per-PoP metrics + cost records |
| Audit | 1 | Immutable audit log |
| **Total** | **22** | Same count as normalized, but with graph + ltree power |

---

## Key Design Decisions

1. **ltree for rule hierarchies** rather than adjacency lists or nested sets. The `tree_path` column enables ancestor queries (`@>`), descendant queries (`<@`), and level-specific queries (`~`) with GIST indexes, all without recursive CTEs. This directly models Akamai's rule tree (up to 5 levels deep) and generalizes to any rule nesting pattern.

2. **Generic graph_edges table** rather than per-relationship junction tables. A single `graph_edges` table with `source_type`/`target_type`/`edge_type` replaces what would otherwise be 8+ junction tables. The JSONB `properties` column stores edge-specific attributes. The trade-off is that foreign key constraints to specific entity tables cannot be enforced at the database level.

3. **Points of Presence (PoPs) as first-class entities.** Unlike other models that treat providers as monolithic, this model explicitly models the geographic distribution of CDN infrastructure. Per-PoP metrics enable queries like "which PoP has the highest error rate for this property?" and graph traversal from country to PoP to provider.

4. **Rule conflict detection via graph edges.** When a new rule is added, the application analyzes path pattern overlaps with existing rules and creates `conflicts_with` graph edges. This makes conflict queries a simple edge lookup rather than an expensive pattern-matching scan.

5. **Blast-radius analysis via recursive edge traversal.** The graph structure enables "if X fails, what is affected?" queries by recursively traversing edges outward from the failing entity. This is critical for multi-CDN platforms where a single origin failure can cascade across multiple properties and providers.

6. **Dual representation for traffic routing.** Traffic targets exist both as relational rows (for CRUD and operational parameters) and as graph edges (`routes_traffic_to`). This supports both simple "list my providers" queries and complex "trace the full traffic path from domain to origin" graph traversals.

7. **Per-PoP metrics** rather than per-provider-only metrics. By linking metrics to specific Points of Presence, the platform can answer geographic performance questions ("which PoP should serve users in Japan?") and inform the ML-based traffic routing model with granular data.
