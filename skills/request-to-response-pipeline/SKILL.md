---
name: request-to-response-pipeline
description: Design, review, or refactor an HTTP API request handler pipeline. Use when checking request processing order, authentication versus authorization, route matching, validation, idempotency, preconditions, domain execution, atomic persistence, response building, headers, telemetry, or side-effect boundaries for any REST/HTTP API.
---

# Request-to-Response Pipeline

Use this skill to design or review a generic HTTP API request handler. The goal
is a regular pipeline: each stage has one responsibility, failures are classified
at the earliest correct point, and irreversible side effects happen only after
all guards have passed.

## Pipeline

Every request passes through this ordered pipeline:

| # | Stage | Responsibility | Typical failure |
| --- | --- | --- | --- |
| 1 | Parse | Decode transport input into normalized request data. | `400` |
| 2 | Authenticate | Verify supplied credentials and establish principal. | `401` |
| 3 | Match route | Resolve method/path to one operation and path parameters. | `404`, `405` |
| 4 | Authorize | Check route policy, scopes, ownership, and tenant boundary. | `401`, `403`, `404` |
| 5 | Validate | Validate media type, headers, query, path params, and body. | `400`, `415`, `422` |
| 6 | Enforce guards | Check idempotency keys, preconditions, and concurrency guards. | `400`, `409`, `412`, `422` |
| 7 | Execute domain operation | Run business rules on validated commands. | `409`, `422` |
| 8 | Persist atomically | Commit mutation, idempotency record, and outbox records together. | `409`, `500`, `503` |
| 9 | Build response | Map domain result to documented transport representation. | `500` |
| 10 | Apply headers | Add cache, tracing, security, representation, and retry headers. | `500` |
| 11 | Emit telemetry/events | Emit logs, metrics, traces, and post-commit events. | no response change |

Keep transport concerns out of domain code. Convert validated HTTP input into a
domain command before executing business rules.

## Review Workflow

1. Identify the framework entry point and the operation's route metadata.
2. Map each behavior to exactly one pipeline stage.
3. Check ordering:
   - parse before authentication
   - route match before route-specific authorization
   - authorization before validation if validation might reveal protected data
   - validation before domain execution
   - idempotency and preconditions before mutation
   - persistence before external publication
   - response building before final headers
   - telemetry after classification
4. Check failure classification against the earliest stage that can know the
   answer.
5. Check side-effect boundaries:
   - no email, webhook, queue publish, payment capture, or success audit before
     atomic persistence
   - external effects use an outbox or equivalent post-commit mechanism
6. Check domain isolation:
   - domain code receives validated commands, not raw framework requests
   - domain code returns domain results or domain errors, not HTTP responses
7. Check idempotency:
   - duplicate-sensitive writes require an idempotency key
   - same key and same fingerprint replay the stored response
   - same key and different fingerprint fail
   - in-progress duplicates are classified explicitly
8. Check preconditions:
   - lost-update risk uses `If-Match` or an equivalent version guard
   - stale versions fail before mutation
9. Check telemetry:
   - one structured request log
   - latency, route, method, status, dependency, and error metrics
   - no secrets, tokens, raw credentials, or unnecessary personal data

## Failure Mapping

Use this default mapping unless a stronger local standard overrides it:

| Problem | Stage | Status |
| --- | --- | --- |
| Malformed JSON, path encoding, or query encoding. | Parse | `400` |
| Expired or invalid bearer token. | Authenticate | `401` |
| Path does not exist. | Match route | `404` |
| Path exists but method is unsupported. | Match route | `405` |
| Missing credentials for private route. | Authorize | `401` |
| Valid principal lacks permission. | Authorize | `403` |
| Resource existence must not be revealed. | Authorize | `404` |
| Unsupported request media type. | Validate | `415` |
| Invalid parameter shape. | Validate | `400` |
| Semantically invalid valid body. | Validate | `422` |
| Missing required idempotency key. | Enforce guards | `400` |
| Same idempotency key is currently processing. | Enforce guards | `409` |
| Same idempotency key has different request fingerprint. | Enforce guards | `422` |
| Stale `If-Match` or equivalent version guard. | Enforce guards | `412` |
| Valid request conflicts with current domain state. | Domain operation | `409` |
| Temporary dependency outage. | Persist | `503` |
| Unexpected implementation defect. | Any stage | `500` |

## Pseudocode Shape

```pseudo
function handle(rawRequest):
  try:
    request = parse(rawRequest)
    principal = authenticate(request)
    route = matchRoute(request)
    authorize(principal, route, request)
    command = validateAndBuildCommand(request, route)

    replay = enforceIdempotencyAndPreconditions(request, route)
    if replay.exists:
      return finalize(replay.response)

    result = executeDomainOperation(command)

    if route.mutatesState:
      transaction:
        persist(result)
        response = buildResponse(result)
        storeIdempotencyResponseIfNeeded(request, response)
        writeOutboxEvents(result.events)
    else:
      response = buildResponse(result)

    return finalize(response)

  catch ClassifiedProblem as problem:
    return finalize(buildProblemResponse(problem))

  catch UnexpectedError:
    return finalize(buildProblemResponse(status = 500))

  finally:
    emitTelemetry()
```

## Output

For reviews, lead with defects in stage order. Each finding should include:

- affected stage
- observed behavior
- expected behavior
- failure/status impact
- minimal correction

For designs, output:

1. stage table for the target API
2. failure classification
3. idempotency/precondition policy
4. side-effect boundary
5. pseudocode only if it clarifies the design
