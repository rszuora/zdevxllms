# Zuora Billing - Agent Guidance

Purpose
Use this file when building resilient integrations with Zuora Billing v1 APIs.

Reliability essentials
- Always log Zuora-Request-Id and processId for support tracing.
- Check success=false in response bodies for POSTs even when HTTP is 200 but know success is not present in responses for GET/list endpoints
- Use Idempotency-Key for POST and PATCH requests.
- Respect Retry-After on 429 and 503.

Correlation and tracing
- Include a client correlation ID (X-Correlation-ID) on all requests.
- Log Zuora-Request-Id and your correlation ID together.
- Log request start/end timestamps and duration for each API call.

Retryable vs non-retryable
- Do not retry on validation errors (missing/invalid fields).
- Retryable: 408, 429, 502, 503, network timeouts, transient DNS errors
- Conditionally retryable: 409 (idempotent conflicts), 5xx other than 501/505
- Non-retryable without change: 400, 401, 403, 404, 422
- Never retry on 400/401/403/404/422 without changing the request.
- Use exponential backoff with jitter
- Use circuit breakers to pause retries during sustained 5xx/429

Retry policy (recommended defaults)
- Max attempts: 5
- Backoff: exponential with full jitter (base 500ms, max 30s)
- Total retry budget per request: 2 minutes

Idempotency
- Generate a unique UUID per business operation.
- Reuse the same Idempotency-Key on retries.
- A 409 can indicate the original request succeeded; read the response body.

Logging and security
- Never log card numbers, CVV, tokens, or secrets.
- Redact PII in logs.
- Store OAuth credentials in a secrets manager.

Structured logging fields
- request_id (Zuora-Request-Id)
- correlation_id (X-Correlation-ID)
- process_id (if present)
- attempt, status_code, latency_ms

Rate and concurrency limits
- Read Rate-Limit-* and Concurrency-Limit-* headers.
- Alert when headroom is low.
- Throttle upstream to avoid bursting after backoff.

Canonical references
- Error recovery: https://developer.zuora.com/docs/guides/error-codes/
- Idempotency: https://developer.zuora.com/docs/guides/idempotent-requests/
- Rate limits: https://developer.zuora.com/docs/guides/rate-limits/
- OpenAPI: https://developer.zuora.com/yaml/zuora-openapi-for-otc.yaml

LLM integration checklist
- Confirm tenant features (Orders, Invoice Settlement, Payments, Revenue) before using gated fields.
- Use only canonical sources (OpenAPI + API reference); do not invent fields, enums, or endpoints.
- Externalize Base URL and credentials; never hardcode secrets.
- Require Zuora-Version only when needed; otherwise use tenant default.
- Use Idempotency-Key on POST/PATCH and treat 409 as possible success.
- Handle success=false in 200 responses; surface reasons[] and processId.
- Respect Rate-Limit-* and Retry-After headers; apply exponential backoff with jitter.
- Redact PII and payment data in logs; do not log PAN/CVV/tokens.
- Prefer Orders API for subscription changes unless explicitly told otherwise.
- Ask for missing identifiers (accountKey, subscriptionId, productRatePlanId) instead of guessing.
