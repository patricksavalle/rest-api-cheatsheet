---
name: rest-api-review
description: Review, design, or rewrite REST/OpenAPI contracts against the REST API Standard. Use when checking API quality, least-surprise resource design, OpenAPI profile compliance, endpoint minimality, status codes, errors, pagination, async jobs, webhooks, security, or compatibility.
---

# REST API Review

Use this skill to design or review HTTP JSON APIs against the repository's REST
API Standard.

## Load Order

1. Read `README.md` for the prose standard.
2. If an OpenAPI file exists, inspect it before proposing changes.
3. Use `openapi-profile/components.yaml` for reusable schemas, parameters,
   headers, responses, and security schemes.
4. Use `openapi-profile/spectral.yaml` for deterministic lint expectations.
5. Use `openapi-profile/example-api.yaml` only as a shape example.

## Review Workflow

1. Identify the API's concrete consumer use cases and domain resources.
2. Check whether each endpoint is necessary; remove aliases and UI-shaped routes.
3. Verify URL regularity:
   - major version prefix, except `/health`
   - plural resource nouns
   - lowercase kebab-case literal segments
   - lowerCamelCase path parameters
   - no key-type segments such as `/orderid/{id}`
   - no imperative actions such as `/cancel`
4. Verify method semantics:
   - `GET` reads only
   - `POST` creates resources or starts operations
   - `PUT` replaces complete resources
   - `PATCH` declares JSON Merge Patch or JSON Patch
   - `DELETE` is idempotent
5. Verify response behavior:
   - minimal standard status-code set
   - `201` includes `Location`
   - `202` includes operation `Location`
   - all `4xx` and `5xx` responses use Problem Details
6. Verify collections:
   - bounded pagination
   - cursor token for mutable or large collections
   - documented sorting, filtering, and sparse fieldsets
7. Verify safety:
   - `ETag` and `If-Match` for lost-update risk
   - `Idempotency-Key` for duplicate-sensitive writes
   - explicit `Cache-Control`
8. Verify security:
   - HTTPS assumption
   - OAuth/OIDC or bearer token scheme
   - scopes per operation
   - no secrets or personal data in URLs
   - CORS only when browser access is required
9. Verify async and webhooks:
   - long-running work is a job resource
   - polling returns `200` job state, not `102`
   - webhooks are registered, signed, replay-protected, retryable, and idempotent
10. Verify compatibility:
   - additive response changes only for compatible releases
   - breaking changes require a new major URL version
   - deprecated endpoints declare removal policy

## Deterministic Checks

When Spectral is available, run the profile ruleset:

```sh
npx spectral lint -r openapi-profile/spectral.yaml path/to/openapi.yaml
```

Treat lint failures as defects. Then perform the judgment review above; linting
cannot determine whether the resource model is minimal, cohesive, or domain-fit.

## Output

For reviews, lead with findings ordered by severity and include exact OpenAPI
paths or operation IDs. Then list missing enforcement or contract gaps.

For designs, output:

1. Resource model.
2. Endpoint table.
3. Required schemas.
4. Status-code and error policy.
5. Pagination/concurrency/async/security notes.
6. Minimal OpenAPI skeleton if requested.
