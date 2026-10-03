# AnatoliaCore Public API

The stable base URL is `https://console.anatoliacore.com/api/public/v1`. The machine-readable contract is published at `/api/public/openapi.json`.

## Authentication and least privilege

Send the API key in `X-API-Key`. Never place it in a URL or request body. Create separate keys per workload, grant only the required scopes, optionally bind each key to one or more source CIDRs, set an expiry, and rotate it regularly.

## Idempotency and operations

Every mutation requires a unique `Idempotency-Key` of at least 16 characters. Reusing the same key with the same canonical request replays the recorded result; reusing it with different input returns `409 idempotency_key_reused`.

Mutation responses include:

- `X-Operation-ID`: durable operation identifier.
- `Operation-Location`: tenant-scoped operation status URL.
- `Idempotent-Replayed: true`: response came from the operation ledger.

If an operation worker disappears after an external side effect, the ledger returns `operation_outcome_unknown` instead of risking the same side effect twice. Inspect the target resource before submitting a new request.

`GET /operations/{id}` returns the durable status, progress, retryability,
resource link, and cancellation state. An operation can be cancelled with
`POST /operations/{id}/cancel` only while it is still pending and before any
external side effect. All later cancellation attempts return
`409 operation_not_cancellable`.

Secrets created by webhook endpoints are returned once and are intentionally excluded from the operation ledger. A replay returns `sensitive_response_not_replayable`; rotate the secret if the original response was lost.

## Pagination and errors

List endpoints accept `page` and `per_page` and return `X-Total-Count`, `X-Page`, `X-Per-Page`, `X-Total-Pages`, and RFC 8288-style `Link` headers.

Errors remain backward-compatible through `detail` and also expose a stable envelope:

```json
{
  "detail": "Resource not found",
  "error": {
    "code": "not_found",
    "message": "Resource not found",
    "request_id": "request-correlation-id",
    "retryable": false
  }
}
```

Only retry when `retryable` is true. Honor `Retry-After` on `429` and in-progress operation responses. Validation errors never echo submitted secret values.

## Clients

- Python SDK and CLI: `sdk/python`
- TypeScript SDK: `sdk/typescript`
- Terraform provider: `terraform-provider-anatoliacore`

All clients require HTTPS, refuse redirects, support request timeouts, understand the stable error envelope, and generate or accept idempotency keys for mutations.

The Terraform provider supports instance, VDC, volume, floating IP, Edge
Firewall security group, and backup policy lifecycle and import. Python and
TypeScript SDKs include operation polling and timestamped HMAC webhook
verification helpers.

## Compatibility and releases

The checked-in public OpenAPI golden contract is validated in CI. Existing v1
paths and methods cannot be removed silently, and mutation operation headers
remain mandatory. SDK and provider release workflows generate SBOMs and GitHub
provenance attestations; release tags and package versions must match.
