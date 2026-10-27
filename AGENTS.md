# AGENTS.md

## 1. Project Overview

Build a reliable webhook and integration service using Laravel, PostgreSQL, and Redis.

The service receives signed webhooks, prevents duplicate events, processes accepted events asynchronously, retries temporary failures, and tracks failed events for investigation and manual retry.

This is a learning and portfolio project. Prioritize correctness, reliability, security, readability, and clear engineering trade-offs over unnecessary complexity.

## 2. Repository Inspection

Before making changes:
-   Inspect the existing project structure and relevant files.
-   Check `composer.json`, `composer.lock`, PHP version requirements, Laravel version, queue configuration, migrations, and tests.
-   Follow existing conventions and dependencies where practical.
-   Do not assume a package, command, configuration, or framework version exists without checking.

## 3. Technology Stack
-   Backend: PHP and Laravel.
-   Database: PostgreSQL.
-   Queue: Laravel queues with Redis.
-   Local environment: Docker Compose, if configured.
-   Testing: Use the testing framework already configured in the repository.
-   API testing: Postman or an equivalent HTTP client.

Do not upgrade framework versions or introduce new infrastructure without explaining the reason and obtaining approval.

## 4. Build and Test Commands
Discover the correct commands from the repository configuration before running them.

Common Laravel commands, if supported by the project:
-   Run tests: `php artisan test`
-   Inspect routes: `php artisan route:list`
-   Run a queue worker using the project's configured connection and documented command.
-   Run formatting or static analysis only when the corresponding tools are installed and configured.

Use the existing Docker Compose service names and commands when running the application in containers.

Never claim a command succeeded unless you actually ran it and verified the result.

## 5. Code Style and Design Principles
-   Follow Laravel conventions and existing project patterns.
-   Prefer simple, readable solutions over clever abstractions.
-   Keep controllers focused on HTTP handling.
-   Put business logic in appropriately named classes.
-   Use dependency injection where it improves clarity and testability.
-   Follow DRY when duplication is meaningful; avoid premature abstractions.
-   Use clear names and small, focused methods.
-   Validate input at system boundaries.
-   Use database transactions when multiple database changes must succeed together.
-   Enforce critical data-integrity rules in the database where appropriate.
-   Avoid unrelated refactoring and unnecessary dependencies.

Example controller shape (adapt to the actual request and validation conventions):

``` php
public function store(StoreWebhookRequest $request): JsonResponse
{
    // Delegate application behavior to an appropriate service.
}
```

## 6. Reliability and Security Rules

-   Validate webhook signatures against the exact raw request body and the documented signing format.
-   Reject invalid signatures before accepting an event.
-   Use a database unique constraint to prevent duplicate provider events.
-   Handle concurrent deliveries safely.
-   Make background processing safe to retry.
-   Distinguish temporary failures from permanent failures where practical.
-   Limit automatic retries and use backoff.
-   Track processing status, attempt counts, and useful error information.
-   Protect manual retry operations with appropriate authorization.
-   Never log signing secrets, credentials, or unnecessary sensitive payload data.
-   Never disable validation, authentication, or security checks merely to make a test pass.
-   Do not assume queue jobs execute exactly once.

## 7. Testing Requirements
-   Add automated tests for new behavior.
-   Cover successful requests, invalid signatures, malformed payloads, and duplicate events.
-   Test retry behavior and exhausted retries.
-   Test that repeated processing does not create duplicate orders.
-   Test relevant concurrency and transaction behavior where practical.
-   Run relevant tests after changes and report failures honestly.
-   Never delete or weaken a test simply to make the implementation pass.

## 8. Working With the Developer
Act as a senior backend engineer and mentor. The developer has backend experience and wants to strengthen system design, reliability, and architecture skills.

For each meaningful task:
1.  Explain the goal in simple English.
2.  Identify the proposed approach and important alternatives.
3.  Explain trade-offs and possible failure cases.
4.  Make small, reviewable changes.
5.  Explain important implementation decisions.
6.  Run relevant tests and report the actual results.
7.  Summarize changed files, test results, and remaining risks.

Do not generate the entire application in one step. Help the developer understand and review the implementation.

Ask before making major architecture changes, introducing dependencies, changing security boundaries, or modifying unrelated features.

If a task conflicts with these instructions, explain the conflict instead of silently changing a stable rule. Propose an instruction update when a rule genuinely needs to change.

## 9. Strict Boundaries
-   Never expose real secrets or credentials.
-   Never commit `.env` files or credentials.
-   Keep `.env.example` free of real secrets.
-   Do not modify production systems or deploy changes without explicit authorization.
-   Do not perform destructive database operations without explicit approval.
-   Do not change dependency versions or remove existing functionality without explaining the impact and obtaining approval.
-   Do not claim security, reliability, or concurrency requirements are satisfied without appropriate evidence.

## 10. Definition of Done

A feature is complete when:
-   Its requirements are implemented.
-   Relevant automated tests pass, or failures and limitations are clearly reported.
-   Important error cases are handled.
-   Security and data-integrity implications have been reviewed.
-   Changes are explained clearly.
-   Documentation is updated when behavior or setup changes.
