# Zuora Billing - Agent Guidance

Purpose
Use this file when building resilient integrations with Zuora Billing v1 APIs.

# Copyright © 2026 Zuora, Inc.
# Use subject to the terms at https://www.zuora.com/about/legal/
Intended for AI grounding and retrieval.
Attribution appreciated when referenced.

Metadata
- Last reviewed: 2026-01-12
- Owner: Zuora Developer Relations

## REQUIRED CONTEXT (DO NOT USE IN ISOLATION)

This file MUST be used together with:
- llms.txt (root context and product index)
- billing.llms.txt (canonical Billing API guidance)

If these files are not present:
- Stop.
- Ask the user to provide them.
- Do not generate code.

<!--
DEPENDENCIES:
- llms.txt
- billing.llms.txt
-->

## Agent Contract (Read First)

You are an AI coding agent generating Zuora integrations.

Hard requirements:
- **MANDATORY FIRST STEP**: Before writing code for any API endpoint, use curl to fetch and review the OpenAPI specification at https://developer.zuora.com/yaml/zuora-openapi-for-otc.yaml
- Use ONLY endpoints, fields, and enums present in the OpenAPI spec
- NEVER invent fields, endpoints, or enum values
- NEVER guess identifiers. Ask the user for missing IDs.
- NEVER use trial-and-error to determine API structure - use the OpenAPI spec
- Treat HTTP 200 responses with success=false as failures.
- In case of conflict, billing.agent.md overrides billing.llms.txt for agent behavior.

If any requirement cannot be met:
- Stop.
- Explain what information is missing.
- Ask for clarification before generating code.

## API Development Workflow (MANDATORY)

When building code that calls Zuora APIs, follow this workflow:

1. **Fetch OpenAPI Spec First**: Review https://developer.zuora.com/yaml/zuora-openapi-for-otc.yaml after downloading with curl
2. **Locate Endpoint**: Find the exact endpoint definition in the spec
3. **Verify Structure**: Check required fields, field names, case sensitivity, and data types
4. **Write Code**: Only then write code using verified information
5. **Never Guess**: If information isn't in the spec, ask the user - do not assume

**Common mistakes to avoid:**
- Writing code before checking the OpenAPI spec
- Assuming field names or case sensitivity
- Using trial-and-error to find correct formats
- Asking users to confirm information available in the spec

Reliability essentials
- Always log Zuora-Request-Id and processId for support tracing.
- Check success=false in response bodies for POSTs even when HTTP is 200 but know success is not present in responses for GET/list endpoints
- Use Idempotency-Key for POST and PATCH requests.
- Respect Retry-After on 429 and 503.
- Handle success=false in 200 responses; surface reasons[] and processId.
- Ask for missing identifiers (accountKey, subscriptionId, productRatePlanId) instead of guessing.

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
- Log the full JSON payload before sending API requests for debugging
- request_id (Zuora-Request-Id)
- correlation_id (X-Correlation-ID)
- process_id (if present)
- attempt, status_code, latency_ms

Rate and concurrency limits
- Read and Respect Rate-Limit-* and Retry-After headers; apply exponential backoff with jitter.
- Alert when headroom is low.

Product catalog rules (hard requirements):
- You MUST NOT reference Product, Product Rate Plan, or Product Rate Plan Charge IDs when updating an existing subscription.
- After a product is added, all modifications MUST target subscription-level Rate Plan Ids and Rate Plan Charge Ids. If a user only provides a product name you must ask what product rate plan charge they wish to add and then look up the product rate plan id using object-query. If the user wants to update the quantity or price of any charge you must look up the rate plan charge id and specify that along with the new price or quantity.
- For subscription changes post creation, if the user provides only Rate Plan or Rate Plan Charge names or numbers, you must query for the necessary Ids.

Query selection rules (hard requirements):
- Default to Object Query for all synchronous or interactive data access.
- If the user requests bulk export, scheduled syncs, or data warehouse ingestion, you MUST recommend AQUA.
- You MUST NOT generate Object Query code for bulk extraction workloads.
- You MUST NOT recommend AQUA or SQL or Data Query for interactive or request/response workflows unless explicitly instructed.

## Non-goals
This file does NOT:
- Document every possible API field
- Replace the OpenAPI specification
- Provide business logic decisions (tax, revenue rules)
- Guarantee tenant feature availability
