# Standards & API Reference

> Project: CDN & Edge Configuration Manager · Generated: 2026-05-03

## Industry Standards & Specifications

### IETF / HTTP Standards

**RFC 9110 — HTTP Semantics**
- URL: https://datatracker.ietf.org/doc/rfc9110/
- Defines the core semantics of the HTTP protocol shared by all versions (HTTP/1.1, HTTP/2, HTTP/3). Establishes the canonical definitions of request methods, status codes, header fields, and URI handling that CDN configuration managers must correctly model and validate.

**RFC 9111 — HTTP Caching**
- URL: https://datatracker.ietf.org/doc/html/rfc9111
- The authoritative standard for HTTP caching behaviour, superseding RFC 7234. Defines Cache-Control directives, freshness calculation, cache validation, and shared cache semantics. Any CDN configuration manager must validate cache policies against this specification; shared CDN caches are explicitly in scope.

**RFC 9213 — Targeted HTTP Cache Control**
- URL: https://httpwg.org/specs/rfc9213.html
- Defines the CDN-Cache-Control response header field, which allows origin servers to direct caching behaviour specifically at CDN caches without affecting browser caches. Also defines a generic convention for creating cache-tier-targeted header fields. Central to any multi-layer cache configuration tool.

**RFC 9211 — The Cache-Status HTTP Response Header Field**
- URL: https://httpwg.org/specs/rfc9211.html
- Defines a standard response header that CDN and cache intermediaries can use to communicate their cache decisions (hit, miss, stale, bypass) to clients and monitoring tools. Relevant for observability and configuration validation in an edge configuration manager.

**RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport**
- URL: https://datatracker.ietf.org/doc/html/rfc9000
- Defines the QUIC transport protocol: a UDP-based, stream-multiplexing, encrypted transport that underpins HTTP/3. CDN configuration managers must expose QUIC-specific settings including connection migration, initial congestion window, and 0-RTT configuration.

**RFC 9114 — HTTP/3**
- URL: https://datatracker.ietf.org/doc/rfc9114/
- Defines the mapping of HTTP semantics over QUIC. CDN platforms that support HTTP/3 (Cloudflare, Fastly, Akamai, Gcore) require configuration of QUIC/HTTP3-specific parameters; an edge configuration manager must model these alongside HTTP/1.1 and HTTP/2 settings.

**RFC 8941 — Structured Field Values for HTTP**
- URL: https://datatracker.ietf.org/doc/html/rfc8941
- Defines a data model for HTTP header and trailer field values (List, Dictionary, Item). Used by newer HTTP standards including RFC 9213 for structured cache directives. Relevant for parsing and validating modern CDN configuration headers.

**RFC 8555 — Automatic Certificate Management Environment (ACME)**
- URL: https://www.rfc-editor.org/rfc/rfc8555.html
- Defines the ACME protocol for automated TLS certificate issuance and renewal. CDN edge nodes must support ACME for certificate lifecycle management; an edge configuration manager should integrate with ACME-compatible CAs (Let's Encrypt, ZeroSSL) to automate certificate provisioning at scale.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Defines the OAuth 2.0 framework for delegated API access. All major CDN management APIs (Cloudflare, Fastly, Akamai) use token-based authentication derived from OAuth 2.0 patterns. Relevant for the authentication layer of any multi-CDN configuration manager.

### W3C Standards

**W3C Edge Architecture Specification 1.0**
- URL: https://www.w3.org/TR/edge-arch/
- Defines the architecture of surrogate proxy networks (CDNs), including how they interact with origin servers and clients. Provides the foundational model for cache invalidation, Surrogate-Control headers, and the ESI composition model.

**W3C Edge Side Includes (ESI) Language Specification 1.0**
- URL: https://www.w3.org/TR/edge-arch/ (submitted as W3C Note; de-facto standard)
- Defines a markup language for assembling page fragments at the CDN edge. Supported by Akamai, Fastly (partial), and Varnish. An edge configuration manager targeting advanced caching should model and validate ESI template configurations.

**W3C Network Error Logging (NEL)**
- URL: https://www.w3.org/TR/network-error-logging/
- Defines the NEL response header that instructs browsers to collect and report network error events (DNS failures, TCP connection errors, TLS errors) to operator-controlled endpoints. CDN platforms that support NEL surface these logs from the edge; configuration managers should expose NEL policy configuration.

**W3C Trace Context**
- URL: https://www.w3.org/TR/trace-context/
- Defines the traceparent and tracestate HTTP headers for propagating distributed trace context across service boundaries, including CDN edge hops. Edge configuration managers that support observability pipelines should ensure configurations preserve Trace Context headers.

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The de-facto standard for describing REST APIs in a machine-readable format. All major CDN providers (Cloudflare publishes OpenAPI schemas; Bunny.net rebuilt their docs as OpenAPI-spec-driven). An AI-native edge configuration manager should expose its own API as a fully-specified OpenAPI 3.1 document and use provider schemas to validate configurations at design time.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/draft/2020-12
- The schema language used natively within OpenAPI 3.1. Relevant for defining and validating the configuration document structures (cache rules, routing policies, header transforms) that an edge configuration manager produces and consumes.

**HashiCorp Configuration Language (HCL) / Terraform Provider Protocol**
- URL: https://developer.hashicorp.com/terraform/plugin/framework
- De-facto standard for IaC-driven CDN configuration. Cloudflare, Fastly, and Akamai all publish official Terraform providers. An edge configuration manager should either consume Terraform state or generate HCL-compatible configuration as an export format.

### Security & Compliance Standards

**OWASP Top 10 for Web Applications**
- URL: https://owasp.org/www-project-top-ten/
- The canonical reference for web application security risks. CDN WAF and security configuration features (blocking injection attacks, XSS, SSRF, misconfigurations) should be framed against OWASP categories. Relevant for AI-driven security header and WAF policy recommendations.

**TLS 1.3 (RFC 8446)**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- The current TLS standard, required by QUIC/HTTP3 (RFC 9000 mandates TLS 1.3). CDN edge configuration managers must model minimum TLS version policy, cipher suite selection, and ALPN negotiation (for HTTP/1.1, HTTP/2, HTTP/3).

**OpenTelemetry Specification**
- URL: https://opentelemetry.io/docs/specs/
- The CNCF standard for distributed tracing, metrics, and logging. CDN providers increasingly support OTLP log export; OpenTelemetry defines the semantic conventions for HTTP server spans (server.address, http.response.status_code, etc.) that should inform the observability data model of an edge configuration manager.

---

## Similar Products — Developer Documentation & APIs

### Cloudflare

- **Description:** Global CDN and edge platform with 330+ PoPs, Workers serverless compute, WAF, DDoS protection, and a unified configuration API covering nearly 3,000 operations.
- **API Documentation:** https://developers.cloudflare.com/api/
- **OpenAPI Schemas:** https://github.com/cloudflare/api-schemas (published and maintained, updated continuously)
- **SDKs/Libraries:**
  - TypeScript/JavaScript: https://github.com/cloudflare/cloudflare-typescript
  - Python: https://github.com/cloudflare/cloudflare-python
  - Go: https://github.com/cloudflare/cloudflare-go
  - Workers Types (TypeScript): via `wrangler types` command
- **Developer Guide:** https://developers.cloudflare.com/
- **Terraform Provider:** https://developers.cloudflare.com/terraform/
- **Pulumi Provider:** https://www.pulumi.com/registry/packages/cloudflare/api-docs/
- **Standards Compliance:** REST/JSON; OpenAPI 3.1 schemas published; Wrangler CLI (IaC); OAuth 2.0 API tokens
- **Authentication:** Bearer token (API Key or API Token with scoped permissions); OAuth 2.0 via Access for user-facing auth

### Fastly

- **Description:** Programmable CDN with real-time cache purging (150ms surrogate-key propagation), VCL-based configuration, Compute@Edge serverless runtime, and streaming logs.
- **API Documentation:** https://www.fastly.com/documentation/reference/api/
- **Postman Collection:** https://www.postman.com/fastly/fastly-developer-hub/documentation/04544o5/fastly-api
- **SDKs/Libraries:** Official client libraries in multiple languages; see https://www.fastly.com/documentation/developers/
- **Developer Guide:** https://www.fastly.com/documentation/developers/
- **CLI:** https://www.fastly.com/documentation/reference/cli/
- **Standards Compliance:** REST/JSON; API Token scoped authentication (e.g., `purge_select` scope); VCL (Varnish Configuration Language) for cache logic
- **Authentication:** `Fastly-Key` HTTP header carrying a scoped API token; token-per-scope model for granular access control (human vs. automation tokens)

### Akamai (Property Manager API)

- **Description:** Largest CDN by PoP count; Property Manager API provides version-controlled configuration management for CDN properties, EdgeWorkers, WAF, and DDoS settings.
- **API Documentation:** https://techdocs.akamai.com/property-mgr/reference/api-get-started
- **EdgeGrid Authentication Docs:** https://techdocs.akamai.com/developer/docs/authenticate-with-edgegrid
- **SDKs/Libraries:**
  - Python: https://github.com/akamai/AkamaiOPEN-edgegrid-python
  - Go: https://github.com/akamai/AkamaiOPEN-edgegrid-golang
  - Node.js: https://github.com/akamai/AkamaiOPEN-edgegrid-node
- **Developer Guide:** https://techdocs.akamai.com/developer/docs/edgegrid
- **Terraform Provider:** https://registry.terraform.io/providers/akamai/akamai/latest/docs
- **Standards Compliance:** REST/JSON; Akamai-specific EdgeGrid authentication (HMAC-SHA-256 signature scheme); property versioning with immutable activated configurations
- **Authentication:** EdgeGrid — a proprietary HMAC-SHA-256 request signing scheme using client_token, access_token, client_secret, and host from an `.edgerc` credential file

### AWS CloudFront

- **Description:** AWS-native CDN service integrated with the AWS ecosystem; supports Lambda@Edge and CloudFront Functions for edge compute, distributions, cache policies, and origin groups.
- **API Documentation:** https://docs.aws.amazon.com/cloudfront/latest/APIReference/Welcome.html
- **Developer Guide:** https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/sdk-general-information-section.html
- **SDKs/Libraries:**
  - JavaScript/Node.js: https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-cloudfront/
  - Python (Boto3): https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/cloudfront.html
  - Go: https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/cloudfront
  - Java: https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/services/cloudfront/CloudFrontClient.html
  - Rust: https://crates.io/crates/aws-sdk-cloudfront
- **Terraform Provider:** `aws` provider (hashicorp/aws) covers CloudFront resources
- **Standards Compliance:** REST/XML (legacy) and JSON; AWS Signature Version 4 (SigV4) for authentication; IAM-based access control
- **Authentication:** AWS SigV4 (HMAC-SHA-256 over canonical request); IAM roles and policies for fine-grained access control

### IO River

- **Description:** Vendor-agnostic multi-CDN orchestration platform integrating 15+ CDN providers (Cloudflare, Akamai, Fastly, AWS CloudFront, etc.) with AI-driven traffic routing and unified analytics.
- **API Documentation:** https://www.ioriver.io/docs/Products/CDNs%20Orchestration/
- **SDKs/Libraries:** API-first; no public SDK listed as of research date; Terraform provider reportedly in development
- **Developer Guide:** https://www.ioriver.io/docs/ (limited public documentation; enterprise onboarding)
- **Standards Compliance:** REST/JSON; proprietary traffic steering and virtual edge APIs; WebAssembly (Wasm) for portable WAF deployment across CDN vendors (announced March 2026)
- **Authentication:** API token; enterprise SSO; specific auth mechanism not publicly documented

### Gcore

- **Description:** Global CDN with 160+ PoPs; FastEdge serverless edge compute based on WebAssembly/Deno; CDN, DDoS protection, and WAAP services accessible via REST API and Terraform.
- **API Documentation:** https://api.gcore.com/docs/cdn
- **SDKs/Libraries:**
  - Go: https://github.com/G-Core/gcorelabscdn-go
- **Developer Guide:** https://gcore.com/docs/fastedge (FastEdge); https://gcore.com/docs (main portal)
- **Terraform Provider:** Available; see Gcore Terraform documentation
- **Standards Compliance:** REST/JSON; FastEdge applications compiled to WebAssembly; OTLP log export
- **Authentication:** Bearer token (API key); documented in https://api.gcore.com/docs/

### Bunny.net

- **Description:** Cost-efficient CDN with pull/push zones, Edge Scripting (Deno-based serverless runtime with sub-15ms cold starts), image optimizer, and newly rebuilt OpenAPI-spec-driven developer documentation.
- **API Documentation:** https://docs.bunny.net/ (rebuilt 2026 as OpenAPI-spec-driven)
- **SDKs/Libraries:**
  - Edge Script SDK (JavaScript/TypeScript): https://github.com/BunnyWay/edge-script-sdk
  - Storage SDKs: TypeScript, PHP, .NET, Java (via https://docs.bunny.net/)
- **Developer Guide:** https://docs.bunny.net/docs/edge-scripting-overview
- **Standards Compliance:** REST/JSON; OpenAPI-spec-driven API reference; Deno-based edge runtime (V8 isolate, TypeScript-native)
- **Authentication:** API key via `AccessKey` header; zone-specific access keys for scoped pull/push zone operations

### Terraform Registry (CDN Providers)

- **Description:** The Terraform provider registry hosts official CDN provider plugins for Cloudflare, Fastly, Akamai, AWS, and Azure, enabling IaC-driven CDN configuration with HCL.
- **Cloudflare Provider:** https://registry.terraform.io/providers/cloudflare/cloudflare/latest
- **Fastly Provider:** https://registry.terraform.io/providers/fastly/fastly/latest
- **Akamai Provider:** https://registry.terraform.io/providers/akamai/akamai/latest
- **AWS Provider (CloudFront):** https://registry.terraform.io/providers/hashicorp/aws/latest
- **Standards Compliance:** HCL (HashiCorp Configuration Language); Terraform Plugin Framework (Go); state-based resource management
- **Authentication:** Each provider delegates to its underlying CDN API authentication (API tokens, SigV4, EdgeGrid, etc.)

### Pulumi Registry (CDN Providers)

- **Description:** Pulumi's registry provides native and Terraform-bridged CDN provider packages for Cloudflare and AWS CloudFront, supporting TypeScript, Python, Go, C#, Java, and YAML.
- **Cloudflare Provider:** https://www.pulumi.com/registry/packages/cloudflare/api-docs/ (v6.13.0, Jan 2026)
- **Developer Guide:** https://developers.cloudflare.com/pulumi/
- **Standards Compliance:** Apache 2.0 open-source core; supports all languages via Pulumi SDK; Terraform bridge extends coverage to Fastly and Akamai providers
- **Authentication:** Delegates to provider-specific credential methods (API tokens, EdgeGrid, etc.)

---

## Notes

- **Surrogate-Control vs. CDN-Cache-Control:** RFC 9213 (CDN-Cache-Control) is the IETF-standardised successor to the proprietary Surrogate-Control header used historically by Akamai and Varnish. New implementations should prefer CDN-Cache-Control; a configuration manager should support both for compatibility with legacy deployments.

- **Akamai EdgeGrid Authentication:** This is a proprietary HMAC-based scheme that does not follow OAuth 2.0 patterns. Integrations with Akamai require explicit implementation of the EdgeGrid signing algorithm, distinct from the bearer-token patterns used by Cloudflare, Fastly, and Gcore.

- **WebAssembly at the Edge:** IO River's March 2026 announcement of Wasm-based portable WAF deployment signals an emerging standard for vendor-agnostic edge logic. The WebAssembly System Interface (WASI) specification (https://wasi.dev/) may become relevant as a portability layer for edge compute workloads across CDN providers.

- **OpenTelemetry Semantic Conventions for HTTP:** The OTEL semantic conventions for HTTP (https://opentelemetry.io/docs/specs/semconv/http/) define standard attribute names (http.request.method, url.full, server.address, http.response.status_code, etc.) that CDN log export pipelines increasingly align to. An edge configuration manager with built-in observability should adopt these conventions.

- **RFC 7234 (obsolete):** While RFC 9111 replaced RFC 7234 in 2022, many CDN dashboards and documentation still reference RFC 7234 terminology. Both should be supported for backwards compatibility in policy validation tooling.
