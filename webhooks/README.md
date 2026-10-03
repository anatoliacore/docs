# Webhooks

Webhook management is available under `/api/public/v1/webhooks`. Endpoint secrets are displayed once, encrypted with an independent production key domain, and used to sign every delivery.

## Verification

Each request contains `Webhook-Id`, `Webhook-Timestamp`, and `Webhook-Signature`. The signature is `v1=` plus the lowercase HMAC-SHA256 hex digest of:

```text
{Webhook-Timestamp}.{raw request body}
```

Verify the signature with a constant-time comparison, reject timestamps outside a short tolerance such as five minutes, and deduplicate events by `Webhook-Id`.

## Delivery guarantees

Events and delivery rows are durably recorded with operation completion. Delivery is at least once. Consumers must be idempotent. Non-2xx responses, timeouts, and connection errors use bounded exponential retries; permanently exhausted deliveries enter the dead-letter state and can be retried explicitly.

Delivery workers use leases so duplicate queue messages cannot send concurrently. Stale leases are reclaimed after worker failure. Endpoints are automatically disabled after repeated failures.

## Network policy

Only HTTPS on port 443 is accepted. Credentials, URL fragments, redirects, private/reserved/link-local/loopback addresses, and hostnames resolving to any non-public address are rejected. DNS is resolved immediately before delivery and the connection is pinned to a validated public address while TLS certificate verification and SNI continue to use the original hostname.
