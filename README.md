> Part of this **[DevOps Project Template](https://github.com/patricksavalle/devops-project-template)**.

# REST API Standard

Use this standard to design, review, and govern HTTP JSON APIs. The goal is a
complete, minimal, predictable interface: regular resource URLs, standard HTTP
semantics, explicit contracts, stable error handling, safe retries, and clear
compatibility rules.

## Repo contents

- [`README.md`](README.md): human-readable REST API Standard.
- [`openapi-profile/components.yaml`](openapi-profile/components.yaml): reusable
  OpenAPI schemas, parameters, headers, responses, and security schemes.
- [`openapi-profile/spectral.yaml`](openapi-profile/spectral.yaml): Spectral
  ruleset for deterministic OpenAPI checks.
- [`openapi-profile/example-api.yaml`](openapi-profile/example-api.yaml):
  minimal API example following the profile.
- [`request-to-response-pipeline/README.md`](request-to-response-pipeline/README.md):
  expanded request handler pipeline with failure classification and pseudocode.
- [`skills/request-to-response-pipeline/SKILL.md`](skills/request-to-response-pipeline/SKILL.md):
  generic AI-agent skill for request handler pipeline design and review.
- [`skills/rest-api-review/SKILL.md`](skills/rest-api-review/SKILL.md): AI-agent
  skill for REST API design and review.

## Table of contents

- [Repo contents](#repo-contents)
- [Principles](#principles)
- [Contract](#contract)
- [Enforcement artifacts](#enforcement-artifacts)
- [Resource model](#resource-model)
- [URLs](#urls)
- [Methods](#methods)
- [Status codes](#status-codes)
- [Requests and responses](#requests-and-responses)
- [Errors](#errors)
- [Collections](#collections)
- [Concurrency and caching](#concurrency-and-caching)
- [Asynchronous operations](#asynchronous-operations)
- [Webhooks](#webhooks)
- [Security](#security)
- [Compatibility and versioning](#compatibility-and-versioning)
- [Health and observability](#health-and-observability)
- [Implementation pipeline](#implementation-pipeline)
- [Review checklist](#review-checklist)
- [References](#references)

## Principles

A high-quality REST API is:

- **Consumer-first**: designed for external developers, not for one UI.
- **Consistent**: the same concept always has the same shape.
- **Cohesive**: every endpoint belongs to the API's domain purpose.
- **Complete**: all necessary use cases are possible.
- **Minimal**: no endpoint, field, parameter, or mode without a concrete use case.
- **Encapsulated**: no database tables, joins, internal IDs, stack traces, or service
  topology leak into the interface.
- **Documented by contract**: the OpenAPI document is the source of truth.
- **Boring by design**: least surprise beats cleverness.

## Contract

- Define the API in **OpenAPI 3.1** before implementation.
- The contract must include every path, method, request schema, response schema,
  status code, error type, security requirement, parameter, header, and example.
- Validate requests and responses against the contract in tests.
- Fail the build on undocumented endpoints, undocumented response codes, and
  breaking contract changes.
- Use JSON unless a specific endpoint needs another media type.

Default media types:

```http
Content-Type: application/json; charset=utf-8
Accept: application/json
```

Problem responses:

```http
Content-Type: application/problem+json
```

## Enforcement artifacts

This repository includes optional artifacts for applying the standard:

- [`openapi-profile/components.yaml`](openapi-profile/components.yaml): reusable
  OpenAPI schemas, parameters, headers, responses, and security schemes.
- [`openapi-profile/spectral.yaml`](openapi-profile/spectral.yaml): deterministic
  OpenAPI lint rules for the enforceable parts of the standard.
- [`openapi-profile/example-api.yaml`](openapi-profile/example-api.yaml): minimal
  example API using the profile.
- [`skills/rest-api-review/SKILL.md`](skills/rest-api-review/SKILL.md): AI-agent
  skill for subjective review and REST API design judgment.

## Resource model

- Base URLs on domain resources, not implementation tables or UI screens.
- Use nouns for resources.
- Use verbs only when they are represented as resources.
- Keep public IDs opaque. Clients must not infer meaning, type, order, shard, or
  storage location from an ID.

Good:

```http
/v1/orders
/v1/orders/{orderId}
/v1/orders/{orderId}/items
/v1/password-resets
/v1/export-jobs
/v1/orders/{orderId}/cancellations
```

Avoid:

```http
/getOrders
/orders/orderid/{id}
/orders/{id}/cancel
/orders-by-customer
/tbl_order_header
```

## URLs

- Put the major API version in the URL: `/v1`.
- Use plural resource names: `/orders`, not `/order`.
- Use lowercase kebab-case for literal path segments.
- Use lowerCamelCase for JSON fields and query parameters.
- Use descriptive path parameter names: `{orderId}`, not `{id}` when nested.
- Do not put ID type, database key type, format, privacy-sensitive data, secrets,
  tokens, or authorization decisions in URLs.
- Do not use trailing slashes.
- Do not use file extensions such as `.json`.

Canonical collection and item shape:

```http
GET    /v1/orders
POST   /v1/orders
GET    /v1/orders/{orderId}
PUT    /v1/orders/{orderId}
PATCH  /v1/orders/{orderId}
DELETE /v1/orders/{orderId}
```

Nested resources are allowed when the child cannot be understood without the
parent or the parent scopes access:

```http
GET  /v1/orders/{orderId}/items
POST /v1/orders/{orderId}/items
GET  /v1/orders/{orderId}/items/{itemId}
```

Keep nesting shallow. Prefer at most two resource levels after the version.

## Methods

| Method | Meaning | Safe | Idempotent | Typical success |
| --- | --- | --- | --- | --- |
| `GET` | Read one resource or a collection | yes | yes | `200`, `304` |
| `POST` | Create a subordinate resource or start an operation | no | no | `201`, `202`, `200` |
| `PUT` | Replace a complete resource | no | yes | `200`, `204` |
| `PATCH` | Partially update a resource | no | depends | `200`, `204` |
| `DELETE` | Remove a resource | no | yes | `204` |

Rules:

- `GET` must not change server state.
- `PUT` replaces the full resource representation.
- `PATCH` must declare exactly one patch format:
  - `application/merge-patch+json` for JSON Merge Patch.
  - `application/json-patch+json` for JSON Patch.
- `DELETE` must remain idempotent. Repeating a successful delete should not create
  a new side effect.
- Duplicate-sensitive `POST` operations must support `Idempotency-Key`.

## Status codes

Use a small, predictable set.

| Code | Use |
| --- | --- |
| `200 OK` | Successful read, update with body, or action result. |
| `201 Created` | New resource created. Include `Location`. |
| `202 Accepted` | Work accepted but not complete. Include operation URL in `Location`. |
| `204 No Content` | Successful operation with no response body. |
| `304 Not Modified` | Conditional `GET` matched the client's cached representation. |
| `400 Bad Request` | Malformed JSON, invalid syntax, invalid parameter shape, or missing required header. |
| `401 Unauthorized` | Missing, expired, or invalid authentication. Include `WWW-Authenticate`. |
| `403 Forbidden` | Authenticated principal is not allowed to perform the operation. |
| `404 Not Found` | Route or resource does not exist, or existence must not be revealed. |
| `405 Method Not Allowed` | Path exists but method is not supported. Include `Allow`. |
| `409 Conflict` | Request conflicts with current domain state. |
| `412 Precondition Failed` | `If-Match` or another precondition failed. |
| `415 Unsupported Media Type` | Unsupported request `Content-Type`. |
| `422 Unprocessable Content` | Request is syntactically valid but semantically invalid. |
| `429 Too Many Requests` | Rate limit exceeded. Include `Retry-After` when possible. |
| `500 Internal Server Error` | Unexpected server failure. |
| `503 Service Unavailable` | Temporary service or dependency outage. Include `Retry-After` when possible. |

Avoid custom status codes. Avoid using `200` for errors.

## Requests and responses

- Use UTF-8.
- Use RFC 3339 timestamps with timezone, preferably UTC:

```json
{
  "createdAt": "2026-06-06T12:30:00Z"
}
```

- Do not use local datetimes without offset.
- Do not use timezone abbreviations such as `EST`.
- Use ISO 4217 currency codes.
- Use ISO 3166-1 alpha-2 country codes.
- Boolean fields must be positive and unambiguous: `isActive`, not `isNotInactive`.
- Arrays must always be arrays, never `null`.
- Empty collections return `200` with an empty array, not `404`.
- `204` responses must not include a body.

Common headers:

| Header | Direction | Use |
| --- | --- | --- |
| `Authorization` | request | Bearer access token. |
| `Content-Type` | request/response | Media type of the body. |
| `Accept` | request | Media types the client accepts. |
| `Accept-Language` | request | Preferred response language when localized output exists. |
| `Date` | response | HTTP response timestamp. |
| `Location` | response | Created resource or accepted operation URL. |
| `Retry-After` | response | Retry delay for `202`, `429`, or `503`. |
| `ETag` | response | Representation version. |
| `If-None-Match` | request | Conditional read. |
| `If-Match` | request | Conditional update/delete. |
| `Idempotency-Key` | request | Safe retry key for duplicate-sensitive writes. |
| `traceparent` | request/response | Distributed tracing context. |

## Errors

Use Problem Details for all `4xx` and `5xx` responses.

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation failed",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "/v1/orders",
  "errors": [
    {
      "field": "customerEmail",
      "reason": "must be a valid email address"
    }
  ]
}
```

Rules:

- `type` must identify a stable problem type.
- `title` must be stable for the problem type.
- `status` must match the HTTP status code.
- `detail` may vary per occurrence.
- `instance` identifies the request or affected resource.
- Extension fields are allowed, but must be documented.
- Do not expose stack traces, internal exception names, SQL, storage paths,
  secret names, hostnames, or dependency topology.

Recommended validation extension:

```json
{
  "errors": [
    {
      "field": "items[0].quantity",
      "reason": "must be greater than zero"
    }
  ]
}
```

## Collections

Collections must support documented pagination. Cursor pagination is the default
for mutable or large collections.

Request:

```http
GET /v1/orders?status=open&limit=50&cursor=eyJpZCI6IjEyMyJ9&sort=-createdAt,orderNumber&fields=id,status,total
```

Parameters:

| Parameter | Meaning |
| --- | --- |
| `limit` | Maximum items to return. Must have a documented maximum. |
| `cursor` | Opaque continuation token returned by the API. |
| `sort` | Comma-separated field list. Prefix with `-` for descending. |
| `fields` | Sparse fieldset using response field names. |

Response:

```json
{
  "data": [
    {
      "id": "ord_123",
      "status": "open",
      "total": {
        "amount": "42.50",
        "currency": "EUR"
      }
    }
  ],
  "page": {
    "limit": 50,
    "nextCursor": "eyJpZCI6IjEyNCJ9"
  }
}
```

Rules:

- Sort and filter fields must be documented and indexed where needed.
- Cursors are opaque. Clients must not parse them.
- Do not use offset pagination for large or frequently changing collections.
- Do not return unbounded collections.

## Concurrency and caching

Use `ETag` for resources that can be cached or updated concurrently.

Conditional read:

```http
GET /v1/orders/{orderId}
If-None-Match: "v3"
```

Unchanged response:

```http
304 Not Modified
```

Conditional update:

```http
PATCH /v1/orders/{orderId}
If-Match: "v3"
Content-Type: application/merge-patch+json
```

Stale response:

```http
412 Precondition Failed
Content-Type: application/problem+json
```

Rules:

- Require `If-Match` for updates where lost updates matter.
- Include media type and representation version in ETag calculation.
- Use `Cache-Control` deliberately:
  - `no-store` for secrets, tokens, and sensitive user-specific responses.
  - `private` for cacheable user-specific responses.
  - `public, max-age=...` only for responses safe for shared caches.
- Do not rely on cache defaults.

## Asynchronous operations

Model long-running work as an operation or job resource.

Start:

```http
POST /v1/export-jobs
Content-Type: application/json
Idempotency-Key: 8b6f4a6c-7f6f-4a5b-9f7e-31b1ecf7a832
```

Response:

```http
202 Accepted
Location: /v1/export-jobs/job_123
Retry-After: 5
```

Poll:

```http
GET /v1/export-jobs/job_123
```

Running:

```json
{
  "id": "job_123",
  "status": "running",
  "createdAt": "2026-06-06T12:30:00Z"
}
```

Completed:

```json
{
  "id": "job_123",
  "status": "succeeded",
  "resultUrl": "/v1/exports/exp_456"
}
```

Rules:

- Do not use `102 Processing` as a polling response.
- Use `202` to accept work.
- Use `200` to return current job state.
- Job status values must be documented. Recommended values:
  - `queued`
  - `running`
  - `succeeded`
  - `failed`
  - `canceled`

## Webhooks

Use webhooks only when polling is not sufficient.

Subscription resource:

```http
POST /v1/webhook-subscriptions
```

Delivery:

```http
POST https://client.example.com/webhooks/orders
Content-Type: application/json
```

Rules:

- Webhook destinations must be registered and verified before use.
- Include an event ID, event type, event time, and subject resource ID.
- Sign deliveries with one documented mechanism.
- If using HMAC, say `HMAC-SHA256`; do not call it `RS256`.
- Include a timestamp in the signed material to prevent replay.
- Retries must use exponential backoff and stop after a documented limit.
- Receivers must treat duplicate event IDs idempotently.

Example event:

```json
{
  "id": "evt_123",
  "type": "order.created",
  "createdAt": "2026-06-06T12:30:00Z",
  "subject": {
    "type": "order",
    "id": "ord_123"
  }
}
```

## Security

- Require HTTPS.
- Use OAuth 2.0 / OpenID Connect for delegated access.
- Use bearer tokens only in the `Authorization` header.
- Access tokens must expire.
- Validate issuer, audience, expiry, signature, and required scopes.
- Do not put tokens, secrets, passwords, personal data, or session identifiers in
  URLs.
- Enforce authorization after authentication and before domain mutation.
- Use least-privilege scopes.
- Validate every entry-point input.
- Return `401` for missing or invalid authentication.
- Return `403` for authenticated but unauthorized requests.
- For browser-accessed APIs, configure CORS explicitly. Do not use wildcard
  origins with credentials.
- Return `X-Content-Type-Options: nosniff`.
- Do not rely on `X-Frame-Options` for JSON APIs. Use it only for HTML responses
  that can be rendered in a browser.

## Compatibility and versioning

- Use major URL versions: `/v1`, `/v2`.
- Keep minor and patch releases backward compatible.
- Additive response fields are backward compatible.
- Removing fields, renaming fields, changing field meaning, changing required
  request fields, changing error type semantics, and changing authorization
  requirements are breaking changes.
- Clients must ignore unknown response fields.
- Servers must reject unknown request fields only when the contract says strict
  request validation is enabled.
- Deprecations must be documented before removal.
- Deprecated endpoints should include a `Deprecation` header and, when known, a
  `Sunset` header.

## Health and observability

Expose a health endpoint that is uncached and safe to call frequently.

```http
GET /health
Cache-Control: no-store
```

Example:

```json
{
  "status": "healthy",
  "checkedAt": "2026-06-06T12:30:00Z",
  "dependencies": [
    {
      "name": "database",
      "status": "healthy"
    }
  ]
}
```

Rules:

- Health responses must not expose secrets, connection strings, hostnames,
  internal URLs, or detailed dependency topology.
- Emit structured logs, metrics, and traces.
- Track request count, latency, error rate, saturation, dependency latency, and
  status-code distribution per endpoint.
- Propagate trace context with `traceparent`.

## Implementation pipeline

See [`request-to-response-pipeline/README.md`](request-to-response-pipeline/README.md)
for the expanded operational model, failure classification, side-effect
boundaries, and pseudocode.

Process each request in this order:

1. Parse method, path, headers, query, and body.
2. Authenticate.
3. Match route.
4. Authorize.
5. Validate media type, headers, query, and body.
6. Enforce idempotency and preconditions.
7. Execute domain operation.
8. Persist atomically where mutation is required.
9. Build response representation.
10. Apply caching, tracing, and security headers.
11. Emit logs, metrics, and events.

Fail fast at the earliest step that can classify the error.

## Review checklist

- [ ] OpenAPI 3.1 contract exists and is the source of truth.
- [ ] URLs follow `/v1/{resources}/{resourceId}` shape.
- [ ] Resource names are plural nouns.
- [ ] Literal path segments are lowercase kebab-case.
- [ ] JSON fields and query parameters are lowerCamelCase.
- [ ] IDs are opaque and never encode database or key type.
- [ ] All methods use standard HTTP semantics.
- [ ] `PATCH` format is explicitly documented.
- [ ] Status codes use the minimal standard set.
- [ ] All errors use Problem Details.
- [ ] Collections are paginated and bounded.
- [ ] Cursor tokens are opaque.
- [ ] Concurrency-sensitive updates use `ETag` and `If-Match`.
- [ ] Cache behavior is explicit.
- [ ] Duplicate-sensitive writes support `Idempotency-Key`.
- [ ] Async work is represented as a job or operation resource.
- [ ] Webhooks are signed, replay-protected, retryable, and idempotent.
- [ ] OAuth/OIDC validation is explicit.
- [ ] No secrets or personal data appear in URLs or logs.
- [ ] Breaking-change rules are documented and enforced.
- [ ] Contract linting, contract tests, and breaking-change checks run in CI.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
- [RFC 3339: Date and Time on the Internet](https://www.rfc-editor.org/rfc/rfc3339)
- [RFC 6750: OAuth 2.0 Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750)
- [RFC 9700: OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700)
- [RFC 7396: JSON Merge Patch](https://www.rfc-editor.org/rfc/rfc7396)
- [RFC 6902: JSON Patch](https://www.rfc-editor.org/info/rfc6902)
- [RFC 9562: UUIDs](https://www.rfc-editor.org/rfc/rfc9562)
- [OpenAPI Specification 3.1](https://spec.openapis.org/oas/v3.1.1.html)
