# Request-to-Response Pipeline

This document expands the REST API Standard's implementation pipeline into an
operational request handler model. The goal is regular behavior: every request
passes through the same stages, each stage has one responsibility, failures are
classified at the earliest correct point, and mutations happen only after all
preconditions are known.

## Core rule

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

## Pipeline state

Use one request context object that is enriched by each stage. Do not pass raw
framework request objects into domain code.

```text
RequestContext
  rawRequest
  method
  path
  headers
  query
  body
  correlationId
  traceContext
  principal
  route
  routeParams
  permissions
  idempotencyKey
  preconditions
  domainCommand
  domainResult
  response
```

The context separates transport concerns from domain work. Domain code receives
only validated commands and returns domain results or domain errors.

## Stage responsibilities

| # | Stage | Responsibility | Typical failure |
| --- | --- | --- | --- |
| 1 | Parse | Decode transport input into a request context. | `400` |
| 2 | Authenticate | Verify supplied credentials and establish principal. | `401` |
| 3 | Match route | Resolve method/path to an operation and path parameters. | `404`, `405` |
| 4 | Authorize | Check route policy, scopes, ownership, and tenant boundary. | `401`, `403`, `404` |
| 5 | Validate | Validate media type, headers, query, and body against contract. | `400`, `415`, `422` |
| 6 | Enforce idempotency and preconditions | Check retry keys, `If-Match`, and other guards before mutation. | `400`, `409`, `412`, `422` |
| 7 | Execute domain operation | Run pure domain decision logic. | `409`, `422` |
| 8 | Persist atomically | Commit mutation and outbox records in one transaction. | `409`, `500`, `503` |
| 9 | Build response | Map domain result to documented response representation. | `500` |
| 10 | Apply headers | Add cache, tracing, security, and representation headers. | `500` |
| 11 | Emit telemetry and events | Emit logs, metrics, traces, and post-commit events. | no response change |

## Stage details

### 1. Parse

Parse only enough to classify transport shape. This stage must not perform domain
validation or mutate state.

Do:

- Normalize method and path.
- Parse headers case-insensitively.
- Parse query parameters.
- Capture request size and content length.
- Parse JSON only when a body is allowed and present.
- Create or extract correlation and trace context.

Fail with:

- `400 Bad Request` for malformed path encoding, invalid query encoding,
  malformed JSON, duplicate invalid parameters, or body on a method that forbids
  it.
- `413 Content Too Large` may be used when the platform exposes this limit.

### 2. Authenticate

Authentication establishes who, if anyone, is calling. It does not yet decide
whether the matched route requires authentication.

Do:

- Extract `Authorization`.
- Verify token signature, issuer, audience, expiry, and not-before time.
- Extract subject, tenant, scopes, and authentication strength.
- Set an anonymous principal when no credential is supplied and the API has
  public routes.

Fail with:

- `401 Unauthorized` when supplied credentials are malformed, expired, invalid,
  or unverifiable.

If the whole API is private, missing credentials can fail here. If the API has
public routes, missing credentials should be handled at authorization after route
policy is known.

### 3. Match route

Route matching maps method and path to one documented operation.

Do:

- Match the normalized path to a route template.
- Extract path parameters.
- Distinguish unknown path from unsupported method.
- Attach operation metadata from the OpenAPI contract or router table.

Fail with:

- `404 Not Found` when no route template matches.
- `405 Method Not Allowed` when the path exists but the method is unsupported.
  Include `Allow`.

### 4. Authorize

Authorization decides whether the principal may execute the matched operation on
the targeted resource.

Do:

- Enforce route authentication requirement.
- Check scopes, roles, tenant, ownership, and resource policy.
- Use resource existence hiding deliberately. If revealing a resource would leak
  information, return `404` instead of `403`.

Fail with:

- `401 Unauthorized` when authentication is required but no valid principal is
  present.
- `403 Forbidden` when the principal is known but lacks permission.
- `404 Not Found` when existence must not be revealed.

### 5. Validate media type, headers, query, and body

Validation checks the request contract before any operation-specific side effect.

Do:

- Require supported `Content-Type` for requests with bodies.
- Validate `Accept` when the API supports multiple response media types.
- Validate required headers.
- Validate path parameters, query parameters, and body schema.
- Reject unknown request fields when the contract is strict.
- Separate syntax errors from semantic domain errors.

Fail with:

- `400 Bad Request` for missing required headers, invalid parameter shape, invalid
  enum casing in query, or malformed request syntax.
- `415 Unsupported Media Type` for unsupported `Content-Type`.
- `406 Not Acceptable` may be used when content negotiation cannot satisfy
  `Accept`.
- `422 Unprocessable Content` for syntactically valid bodies that violate
  semantic request constraints.

### 6. Enforce idempotency and preconditions

This is the last gate before domain execution and mutation.

Do:

- Require `Idempotency-Key` for duplicate-sensitive `POST` operations.
- Check whether the same idempotency key already has a stored response.
- Reject reuse of the same key with a different request fingerprint.
- Require `If-Match` for operations with lost-update risk.
- Compare `If-Match` with the current resource version.
- Check other route-level preconditions.

Fail with:

- `400 Bad Request` when a required idempotency key or precondition header is
  missing.
- `409 Conflict` when an identical idempotency key is currently processing.
- `412 Precondition Failed` when `If-Match` or another precondition fails.
- `422 Unprocessable Content` when an idempotency key is reused with a different
  request fingerprint.

If a completed idempotency record exists for the same fingerprint, return the
stored response without executing the domain operation again.

### 7. Execute domain operation

Domain execution makes the business decision. Keep it independent from HTTP,
databases, queues, and framework objects.

Do:

- Convert validated transport input into a domain command.
- Load the minimum domain state needed for the decision.
- Run invariants and business rules.
- Produce a domain result plus intended state changes.
- Avoid irreversible side effects.

Fail with:

- `409 Conflict` for current-state conflicts, such as canceling an already
  shipped order.
- `422 Unprocessable Content` for valid input that cannot satisfy domain rules.
- `500 Internal Server Error` only for unexpected failures.

### 8. Persist atomically

Persistence commits mutation. This stage must be atomic for the aggregate or
transaction boundary being changed.

Do:

- Persist state changes in one transaction where mutation is required.
- Store idempotency response records in the same transaction as the mutation.
- Write outbox events in the same transaction as the mutation.
- Use optimistic concurrency where applicable.

Fail with:

- `409 Conflict` for concurrent state conflicts detected by storage.
- `503 Service Unavailable` for temporary dependency outage.
- `500 Internal Server Error` for unexpected persistence failure.

Do not publish external events before the mutation commits.

### 9. Build response representation

Map the domain result to the documented response. This is a representation step,
not a domain step.

Do:

- Select the documented success status code.
- Build response body from the domain model, not storage records.
- Include `Location` for `201` and `202`.
- Include `ETag` when the representation is versioned.
- Use Problem Details for errors.

Fail with:

- `500 Internal Server Error` when the server cannot produce a documented
  response for a valid domain result. Treat this as a contract or implementation
  defect.

### 10. Apply caching, tracing, and security headers

Apply final transport headers consistently.

Do:

- Add `Date`.
- Add `Cache-Control`.
- Add `ETag` for versioned representations.
- Add `Retry-After` for `202`, `429`, and `503` where useful.
- Add `X-Content-Type-Options: nosniff`.
- Propagate or return trace context.
- Add CORS headers only for browser-facing APIs.

Do not add `X-Frame-Options` to JSON API responses unless the endpoint serves
browser-rendered HTML.

### 11. Emit logs, metrics, and events

Telemetry is mandatory, but it must not change the HTTP response after the
response has been classified.

Do:

- Emit one structured access log per request.
- Emit metrics for latency, status code, route, method, request size, response
  size, and dependency latency.
- Finish tracing spans with error classification.
- Publish committed outbox events after persistence succeeds.
- Never log secrets, tokens, full personal data, or raw credentials.

Telemetry failure must not turn a successful business operation into an HTTP
failure. Emit best-effort diagnostics and continue.

## Pseudocode

```pseudo
function handle(rawRequest):
  context = new RequestContext(rawRequest)
  startedAt = clock.now()

  try:
    parseTransport(context)
    authenticate(context)
    matchRoute(context)
    authorize(context)
    validateRequest(context)

    replay = enforceIdempotencyAndPreconditions(context)
    if replay.exists:
      context.response = replay.response
      return finalize(context)

    context.domainCommand = buildDomainCommand(context)
    context.domainResult = executeDomainOperation(context.domainCommand)

    if context.route.mutatesState:
      transaction:
        persistDomainResult(context.domainResult)
        context.response = buildResponseRepresentation(context)
        if context.idempotencyKey exists:
          storeFinalIdempotencyResponse(context.idempotencyKey, context.response)
        writeOutboxEvents(context.domainResult.events)
    else:
      context.response = buildResponseRepresentation(context)

    return finalize(context)

  catch Problem as problem:
    context.response = buildProblemResponse(problem)
    return finalize(context)

  catch UnexpectedError as error:
    context.response = buildProblemResponse({
      status: 500,
      type: "https://api.example.com/problems/internal-server-error",
      title: "Internal server error"
    })
    context.internalError = error
    return finalize(context)

  finally:
    emitTelemetry(context, startedAt)

function finalize(context):
  applyRepresentationHeaders(context)
  applyCacheHeaders(context)
  applyTraceHeaders(context)
  applySecurityHeaders(context)
  return context.response
```

For operations that mutate state and use idempotency, implementations usually
combine domain persistence, outbox writes, and idempotency response storage in
one transaction. If the response cannot be fully known until after commit, store
enough deterministic data to rebuild the same response for retries.

## Failure classification

| Earliest classifier | Example | Response |
| --- | --- | --- |
| Parse | Malformed JSON. | `400` |
| Authenticate | Expired bearer token. | `401` |
| Match route | `/v1/orderz` does not exist. | `404` |
| Match route | `POST` on read-only item path. | `405` |
| Authorize | Missing token for private route. | `401` |
| Authorize | Valid user lacks required scope. | `403` |
| Validate | `Content-Type: text/plain` for JSON endpoint. | `415` |
| Validate | `limit=abc`. | `400` |
| Validate | `quantity` is `0`. | `422` |
| Preconditions | Stale `If-Match`. | `412` |
| Idempotency | Same key still processing. | `409` |
| Idempotency | Same key, different body. | `422` |
| Domain operation | Canceling a shipped order. | `409` |
| Persistence | Temporary database outage. | `503` |
| Response build | Domain result has no documented representation. | `500` |

## Side-effect boundaries

No irreversible side effect may happen before stage 8.

Allowed before persistence:

- parsing
- credential verification
- route lookup
- authorization checks
- validation
- idempotency lookup
- precondition lookup
- domain decision logic

Not allowed before persistence:

- sending emails
- publishing webhooks
- charging payments
- writing audit events that imply success
- changing resource state outside the transaction

Use an outbox for external side effects. The request transaction writes the
outbox record; a separate worker publishes it after commit.

## Review checklist

- [ ] Every stage has one responsibility.
- [ ] Authentication and authorization are separate.
- [ ] Missing credentials are classified after route policy is known, unless the
      whole API is private.
- [ ] Validation happens before mutation.
- [ ] Idempotency and preconditions happen before domain execution.
- [ ] Domain code does not receive raw HTTP framework objects.
- [ ] Mutations and outbox writes are atomic.
- [ ] External events are published only after commit.
- [ ] All failures are represented as Problem Details.
- [ ] Telemetry failure does not change successful API behavior.
