# PROJECT_SPEC.md

## 1. Project Goal
Build a small, reliable webhook service that receives order events from an external store and processes them asynchronously without creating duplicate orders.

This is a portfolio and learning project intended to demonstrate backend engineering fundamentals: API design, signature validation, database integrity, queues, retries, automated testing, and operational visibility.

## 2. Scope
### In scope
-   Receive webhook requests over HTTP.
-   Validate webhook signatures.
-   Validate event payloads.
-   Persist accepted events.
-   Protect against duplicate event deliveries.
-   Process events asynchronously using Laravel queues and Redis.
-   Retry temporary processing failures with bounded backoff.
-   Track event processing status and errors.
-   Support an authorized manual retry for failed events.
-   Provide useful, safe logs.
-   Include automated tests and clear local setup instructions.

### Out of scope for the first version
-   Kubernetes or a multi-service deployment.
-   Kafka or a separate event-streaming platform.
-   A full admin frontend.
-   Support for many real webhook providers.
-   Production deployment and cloud infrastructure.
-   Complex role management beyond what is needed to protect manual retry.
-   Exactly-once delivery guarantees across external systems.

Do not add out-of-scope features without explaining the value and receiving approval.

## 3. Example Use Case
An external store sends an `order.created` event. The service validates the request, records the event, and queues it for processing. A worker creates the corresponding local order.

If the store sends the same event again, the service must not create a second event record or order. If processing fails temporarily, the job should retry. If retries are exhausted, the event should be visible as failed and eligible for an authorized manual retry.

## 4. Technology
-   PHP and Laravel.
-   PostgreSQL as the persistent database.
-   Redis as the queue backend.
-   Docker Compose for local development if the repository already uses it.
-   The test framework already installed and configured in the repository.

Before implementation, inspect the repository and record the actual PHP, Laravel, PostgreSQL, Redis, and testing-framework versions in the README or architecture notes. Do not guess versions.

## 5. Webhook Contract
Initial endpoint:

`POST /api/webhooks/store`

Example headers:
-   `Content-Type: application/json`
-   `X-Provider-ID: demo-store`
-   `X-Event-ID: evt_1001`
-   `X-Event-Timestamp: <unix timestamp>`
-   `X-Webhook-Signature: sha256=<signature>`

Example body:

``` json
{
  "type": "order.created",
  "data": { "order_id": "ORD-1001", "amount": 250000, "currency": "IDR"
  }
}
```

This is an example contract, not a claim that a real provider uses these exact headers. Document the project's signing format and implement a test helper or fake sender that produces valid signatures.

Signature validation must use the exact raw request body and the agreed signing format. Include a timestamp freshness check if the signing scheme uses timestamps. Keep secrets in environment configuration, never in source control.

## 6. Expected HTTP Behavior
-   Valid, newly accepted event: return `202 Accepted` after the event is safely persisted and queueing behavior is handled according to the chosen design.
-   Invalid signature: return `401 Unauthorized`.
-   Malformed JSON or invalid required fields: return `400 Bad Request` or the consistent validation status chosen and documented by the project.
-   Duplicate event with the same provider and event ID: return a documented successful duplicate response without processing it twice.
-   Same provider and event ID with a different payload: detect and document the conflict; do not silently treat it as an ordinary duplicate.
-   Unexpected server failure before safe acceptance: return an appropriate server error and avoid claiming the event was accepted.

The API response must not imply that asynchronous order processing has already completed.

## 7. Idempotency and Data Integrity
-   Identify an event using the provider identity plus the provider's event ID.
-   Enforce uniqueness in PostgreSQL with a unique constraint.
-   Handle concurrent duplicate deliveries safely; an application-level pre-check alone is insufficient.
-   Make order creation safe if a job is retried or delivered more than once.
-   Use transactions where needed to keep related database changes consistent.

## 8. Processing and Retry Behavior
-   Persist accepted events before treating them as accepted.
-   Dispatch work to a Laravel queue backed by Redis.
-   Track processing status and attempt count.
-   Begin with a bounded retry policy, such as three total attempts, with increasing delays. Confirm the exact meaning of "attempts" in the Laravel configuration and document it.
-   Retry temporary failures; avoid retrying known permanent validation or business-rule failures indefinitely.
-   After retries are exhausted, mark the event as failed and retain a safe, useful error summary.
-   Provide a protected way for an authorized operator to retry a failed event.
-   Manual retry should reset or schedule the event safely without creating duplicate orders.

The implementation must account for the failure window where an event is committed to the database but queue dispatch fails. Document the initial solution and its limitations. Consider a transactional outbox as a later improvement, not a prerequisite for the first version.

## 9. Event Status and Observability
Use a small, documented state model. Possible states include:

-   `received`
-   `queued`
-   `processing`
-   `processed`
-   `failed`

Adjust the states if the chosen Laravel queue lifecycle makes another model clearer. Define when each transition occurs.

Track, at minimum, the provider, external event ID, event type, status, attempt count, received time, processed time when applicable, and a safe last-error summary. Use structured logs where practical and include a stable event identifier for tracing.

Do not log secrets or unnecessary personal or payment data.

## 10. Manual Retry and Access Control
Provide a documented endpoint or command for retrying failed events. It must not be publicly usable without appropriate authorization. For local development, document the selected authentication or operator-only approach rather than leaving a production-sensitive endpoint open.

Record the retry action and outcome in logs.

## 11. Automated Tests
At minimum, test:

1.  A valid signature is accepted.
2.  An invalid signature is rejected.
3.  A malformed payload is rejected.
4.  A duplicate event does not create a second event or order.
5.  Concurrent duplicate deliveries cannot create duplicate orders, where the test environment supports a meaningful concurrency test.
6.  A temporary processing failure is retried and can eventually succeed.
7.  A permanently failing job is marked failed after the configured retry limit.
8.  A failed event can be retried through the authorized mechanism.
9.  Reprocessing an event does not create duplicate orders.
10. Queue-dispatch or persistence failure behavior matches the documented design.

Use fakes or deterministic test hooks for failure simulation. Do not add an insecure, publicly accessible failure switch to production behavior.

## 12. Acceptance Criteria
The first version is complete when:

-   A developer can run it locally using the documented setup.
-   Signature validation and payload validation work.
-   Duplicate and concurrent deliveries are protected by database integrity and application behavior.
-   Events are processed asynchronously.
-   Temporary failures retry with a bounded policy.
-   Exhausted failures are visible and can be retried by an authorized operator.
-   Logs and stored status make event outcomes understandable.
-   The required automated tests pass.
-   The README documents setup, sample requests, test commands, retry behavior, and known limitations.

## 13. Decisions That May Evolve
Keep these decisions documented and revisit them when evidence warrants it:

-   Exact event status values.
-   Exact retry count and backoff intervals.
-   How to handle same-ID/different-payload conflicts.
-   The manual retry interface and authorization mechanism.
-   Whether to add a transactional outbox.
-   Whether a simple event log is sufficient or more structured observability is needed.

Changes should be made deliberately and reflected in tests and documentation.
