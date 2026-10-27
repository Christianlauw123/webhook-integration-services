# ARCHITECTURE.md

## 1. Status
This document describes the intended initial architecture. Validate it against the actual repository and update it as implementation decisions are made. Do not treat proposed choices as implemented until verified.

## 2. Goals and Trade-offs
Goals:
-   Accept valid webhooks quickly and safely.
-   Reject unauthenticated or malformed requests.
-   Avoid duplicate events and duplicate order creation.
-   Process work asynchronously.
-   Retry temporary failures and expose exhausted failures.
-   Keep the design small enough to understand and explain.

Trade-offs:
-   A single Laravel application is simpler to build and operate than separate microservices.
-   PostgreSQL is the source of truth for events and orders; Redis is used for queue transport, not as the only record of an event.
-   Laravel queues provide practical background processing, but do not guarantee that a job will execute exactly once.
-   The first version may use a simpler queue-dispatch approach; a transactional outbox can reduce the risk of a database/queue dual-write gap later.

## 3. High-Level Components

1.  **External store / fake sender**: sends an event, timestamp, and signature.
2.  **Webhook route and controller**: receives the HTTP request and coordinates boundary validation.
3.  **Signature verifier**: validates the signature using the exact raw request body and configured secret.
4.  **Payload validation**: checks required fields and supported event types.
5.  **Event persistence**: stores the provider event with a unique key.
6.  **Queue dispatch**: schedules a background job using Laravel queues and Redis.
7.  **Event processing job/service**: applies business logic and creates the order safely.
8.  **PostgreSQL**: stores webhook event records and business orders.
9.  **Failure and retry interface**: allows an authorized operator to inspect and retry failed events.
10. **Logs and tests**: help explain event outcomes and verify reliability behavior.

## 4. Main Request Flow

1.  Receive the webhook request.
2.  Read the exact raw body and required headers.
3.  Validate timestamp freshness when required by the signing scheme.
4.  Verify the signature using a constant-time comparison where supported by the language/runtime.
5.  Validate the JSON structure and required business fields.
6.  Attempt to insert the event using a unique key based on provider plus external event ID.
7.  If the event already exists, compare the stored payload identity or hash to detect an ordinary duplicate versus an ID/payload conflict.
8.  Safely arrange asynchronous processing for the accepted event.
9.  Return a response that indicates acceptance, not completed business processing.

Never rely solely on "check whether it exists, then insert." Two concurrent requests can both pass the check. The database unique constraint is the final protection.

## 5. Background Processing Flow

1.  A Laravel worker receives the queued job.
2.  The job marks or records the event as processing, according to the selected state model.
3.  The processor validates any remaining business rules.
4.  It creates the order in a database transaction, using an appropriate idempotency guard.
5.  On success, it marks the event processed and records the processing time.
6.  On a retryable failure, the queue retry policy schedules another attempt.
7.  When the retry limit is exhausted, the event is marked failed and a safe error summary is recorded.
8.  An authorized operator can request a manual retry.

Keep event-processing logic in a focused service or job rather than embedding all behavior in the controller.

## 6. Data Model (Initial Proposal)
### `webhook_events`
Potential columns:
-   `id`
-   `provider`
-   `external_event_id`
-   `event_type`
-   `payload` (JSON/JSONB, subject to privacy and retention needs)
-   `payload_hash` (optional; useful for detecting changed payloads for the same event ID)
-   `status`
-   `attempts`
-   `last_error`
-   `received_at`
-   `processed_at`
-   timestamps

Constraints and indexes:
-   Unique constraint on `(provider, external_event_id)`.
-   Indexes for status and received time if needed for failed-event queries.
-   Store only the payload data needed for processing and debugging; define retention if payloads can contain sensitive data.

### `orders`
Potential columns:
-   `id`
-   `external_order_id`
-   `provider`
-   `amount`
-   `currency`
-   timestamps

Add a suitable unique constraint for the external order identity, such
as `(provider, external_order_id)`, if that matches the business
contract. This provides a second layer of protection against duplicate
order creation.

This is a proposed schema. Review data types, money representation, nullability, indexes, and relationships before implementing migrations.

## 7. Idempotency Strategy
Use multiple safeguards:
-   Event-level uniqueness: `(provider, external_event_id)`.
-   Payload comparison or hash to detect conflicting deliveries that reuse an event ID with different content.
-   Business-level uniqueness for the external order identity where appropriate.
-   Transactions and conflict-safe database operations.
-   Tests that deliver the same event repeatedly and, where practical, concurrently.

A queue retry can run the same job more than once. Correctness must not depend on the queue executing it only once.

## 8. Signature and Replay Protection
-   Define a clear signing format for the fake sender, including how the timestamp and raw body are combined.
-   Calculate the expected HMAC using the configured secret.
-   Compare signatures safely using a constant-time comparison function where available.
-   Reject stale timestamps if the protocol includes timestamp-based freshness.
-   Keep the secret in environment configuration.
-   Do not log the secret or full signature inputs.

A timestamp freshness check helps reduce replay risk, but idempotency is still needed because valid requests can be delivered repeatedly.

## 9. Queue and Retry Design
Initial direction:
-   Use Laravel queues with Redis.
-   Configure a bounded number of attempts and increasing retry delays.
-   Decide explicitly which errors are retryable.
-   Keep job handling idempotent.
-   Record the final failure in the application database, not only in transient logs.

Confirm Laravel's `tries`, `backoff`, timeout, failed-job, and queue connection behavior against the actual installed framework version.

### Database-to-queue failure window
If the application commits the event to PostgreSQL and then fails before queue dispatch succeeds, the event may remain stored without being processed.

For the first version, choose and document a pragmatic recovery mechanism, such as a safe redispatch path or a command that finds eligible unqueued events. Ensure redispatch is idempotent.

A later improvement is the **transactional outbox**: write the event and an outbox record in the same database transaction, then have a dispatcher publish pending outbox records to the queue. This narrows the dual-write failure gap but adds implementation complexity.

## 10. Event Status Model
Possible states:
-   `received`: validated and persisted.
-   `queued`: work has been scheduled.
-   `processing`: a worker has started processing.
-   `processed`: business processing completed successfully.
-   `failed`: automatic retries were exhausted or a non-retryable failure was recorded.

Define state transitions carefully. Queue dispatch and worker execution can race, so do not assume a status update always happens in a simple linear order. If the chosen implementation uses fewer states, document why.

## 11. Manual Retry and Operations
-   Restrict manual retry to an authorized operator or a protected local-only command.
-   Retry only eligible failed events.
-   Avoid clearing useful diagnostic information; preserve retry history where practical.
-   Record who/what triggered a retry when identity is available.
-   Ensure a manual retry cannot create a duplicate order.
-   Return useful status information without exposing secrets or sensitive payloads.

## 12. Observability
Use structured logs where practical. Include a stable event identifier and useful fields such as provider, event type, attempt number, and status transition.

Track enough information to answer:
-   Was the event received?
-   Did signature validation pass?
-   Was it accepted as new or recognized as a duplicate?
-   Was it queued and processed?
-   How many attempts occurred?
-   Why did it fail?
-   Was it retried manually?

Do not log credentials, signing secrets, or unnecessary personal/payment data.

## 13. Testing Strategy
Test at the right level:
-   Feature/API tests for request validation, signature handling, and response codes.
-   Database-backed tests for uniqueness and order creation.
-   Queue fakes or queue integration tests for dispatch behavior.
-   Job/service tests for successful processing, retryable errors, and permanent failures.
-   Tests for manual retry authorization and idempotency.
-   Concurrency tests where the test infrastructure can exercise real concurrent requests meaningfully.

Avoid tests that only verify internal method calls when the important requirement is an externally observable behavior.

## 14. Known Limitations to Document
Before calling the project portfolio-ready, document:
-   Whether the queue-dispatch gap is fully addressed or only mitigated.
-   What the retry count means and which failures are retried.
-   How the signing format and timestamp freshness work.
-   How duplicate IDs with different payloads are handled.
-   What authorization protects manual retry.
-   Payload retention and sensitive-data handling.
-   Which concurrency behaviors were tested and which remain unverified.

## 15. Decisions Log
When a meaningful architecture decision changes, record:
-   Decision and date.
-   Context/problem.
-   Options considered.
-   Chosen approach.
-   Trade-offs and follow-up work.

Keep this document aligned with the implementation. Do not leave proposed designs described as if they already exist.
