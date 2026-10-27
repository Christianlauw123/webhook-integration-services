# Reliable Webhook & Integration Service

A small, reliable webhook and integration service built with Laravel, PostgreSQL, and Redis.

The service receives signed webhooks, prevents duplicate events, processes accepted events
asynchronously, retries temporary failures, and tracks failed events for investigation and
authorized manual retry.

This is a learning and portfolio project. It prioritizes correctness, reliability, security,
and readability over unnecessary complexity.

## Project documents

| Document | Purpose |
|---|---|
| [PROJECT_SPEC.md](./PROJECT_SPEC.md) | Requirements, scope, webhook contract, acceptance criteria |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Intended design, data model, idempotency and retry strategy |
| [AGENTS.md](./AGENTS.md) | Working rules and conventions for this repository |

Per PROJECT_SPEC section 4, this README records the **actually verified** versions rather than
guessed ones.

## Current status

**Early scaffold.** Only the stock Laravel skeleton exists. No webhook logic, no signature
validation, no migrations, and no queue configuration have been implemented yet.

Last verified: **2026-10-28**.

### What works today

- Laravel boots and serves: `php artisan --version`, `php artisan route:list`
- The default test suite passes: **2 tests, 2 assertions, 0 failures**
- `APP_KEY` is generated locally
- Git repository is initialized (branch `main`); `.env` and `vendor/` are ignored

### What does not work yet

- **Database access fails.** `.env` still uses Laravel's stock `DB_CONNECTION=sqlite`, but the
  `pdo_sqlite` extension is not enabled in the local PHP build, and the project requires
  PostgreSQL anyway. `php artisan migrate:status` fails with `could not find driver`.
- **No queue backend is running.** `.env` uses `QUEUE_CONNECTION=database`; Redis is not
  installed locally and nothing is listening on port 6379.
- No webhook endpoint, signature verification, or event persistence exists.
- The local `.env` has not been switched from SQLite to PostgreSQL.

## Verified environment versions

Recorded on 2026-10-28 by running the commands shown. These are observed values, not
assumptions.

| Component | Version | How it was verified |
|---|---|---|
| PHP | **8.5.12** (CLI, ZTS, VS17, x64) | `php -v` |
| Composer | **2.9.8** | `composer --version` |
| Laravel framework | **13.39.0** | `php artisan --version` |
| Laravel skeleton | `laravel/laravel` package name in `composer.json` | `composer.json` |
| PHPUnit | **12.5.42** | `composer.lock` |
| Laravel Pint | 1.32.1 | `composer.lock` |
| PostgreSQL | **18.2** (server and `psql`) | `psql --version`, `pg_isready` |
| Redis | **not installed** (nothing listening on 6379) | `netstat` |

`composer.json` requires `php: ^8.3` and `laravel/framework: ^13.17`. The local PHP 8.5.12
satisfies both, and the test suite runs without deprecation warnings.

### Enabled PHP extensions

`bcmath, ctype, curl, dom, fileinfo, filter, gd, hash, mbstring, openssl, pdo_pgsql, pgsql,
tokenizer, xml, zip` (plus PHP built-ins).

Notably **missing**: `pdo_sqlite` and `redis`. Both matter for local setup; see the notes below.

## Local setup

### Prerequisites

- PHP 8.3 or newer with `pdo_pgsql` and `openssl` enabled
- Composer
- PostgreSQL
- Redis (required for the queue; not yet configured in this repository)

### Install

```sh
composer install
cp .env.example .env
php artisan key:generate
```

### Configure the database

Edit `.env` and point it at PostgreSQL. Do not commit `.env`.

```env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=webhook_integration
DB_USERNAME=your_username
DB_PASSWORD=your_password
```

Create the database, then run migrations once they exist:

```sh
php artisan migrate
```

> **Note:** the stock `.env.example` currently ships `DB_CONNECTION=sqlite`. This has not yet
> been changed to PostgreSQL, and `pdo_sqlite` is not enabled locally, so database commands
> fail until `.env` is updated.

### Configure the queue

Per ARCHITECTURE.md, queues use Redis:

```env
QUEUE_CONNECTION=redis
REDIS_CLIENT=phpredis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

A Redis server must be running, and a Redis client driver installed. `phpredis` is a native
extension; `predis/predis` is a pure-PHP alternative that requires no extension. This choice
has not been finalized.

## Test commands

```sh
php artisan test                                  # run the full suite
php artisan test --filter=NameOfTest              # run a focused test
php artisan route:list                            # inspect registered routes
php artisan migrate:status                        # inspect migration state
vendor/bin/pint --test                            # check code style
```

The default `phpunit.xml` points tests at SQLite (`DB_CONNECTION=sqlite`, `DB_DATABASE=:memory:`).
Because `pdo_sqlite` is not enabled locally, that setting will need to change to PostgreSQL
before database-backed tests can run.

## Security notes

- Never commit `.env` or real credentials. `.env` is already listed in `.gitignore`.
- Keep webhook signing secrets in environment configuration only.
- Do not log signing secrets, signatures, or unnecessary payment and personal data.
- The manual retry mechanism must be protected by authorization before it is exposed.

## Known limitations

To be completed as the implementation progresses, per ARCHITECTURE.md section 14:

- Whether the database-to-queue dispatch gap is fully addressed or only mitigated
- What the retry count means and which failures are retried
- How the signing format and timestamp freshness work
- How duplicate event IDs with different payloads are handled
- What authorization protects manual retry
- Payload retention and sensitive-data handling
- Which concurrency behaviors were tested and which remain unverified

## Next steps

1. Point `.env` at PostgreSQL and confirm `php artisan migrate` succeeds.
2. Define and document the webhook signing format.
3. Add `webhook_events` and `orders` migrations with the required unique constraints.
4. Implement signature verification and payload validation.
5. Implement queued processing with bounded retries.
6. Add the required automated tests.