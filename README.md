# InMediam Fullstack Challenge - Plan subscription API

A Laravel 9 API for subscribing to a plan with a simulated payment and switching plans, where the unused days of the current plan become credit.

## Context

Solution to the hiring challenge of InMediam (2025, "Desafio Fullstack"). This repository started as a fork of [`InMediam/desafio-fullstack`](https://github.com/InMediam/desafio-fullstack). That template provides a Laravel API with plans and a single user, a React + Vite front-end scaffold, and a PostgreSQL `docker-compose.yml`. The author's work starts at commit `8ae6dc8` (2025-07-15). Everything before it is the template.

### What came from the template

- `front/`: the React 18 + Vite + TypeScript + Tailwind scaffold. **It is unchanged.** The only addition is an empty `front/.env` (`VITE_API_URL=`).
- `db/docker-compose.yml`: a PostgreSQL container.
- The Laravel 9 skeleton in `api/`, including the `plans` and `users` migrations, `PlanSeeder` (6 plans), `UserSeeder` (1 user), the `Plan` and `User` models, `GET /api/plans` and `GET /api/user`.
- The challenge statement, kept in Portuguese in [`docs/challenge.md`](docs/challenge.md).

### What the author built (all in `api/`)

- **Migrations** `contracts` (`user_id`, `plan_id`, `is_active`, `value`, `discount`, `started_at`, `ended_at`) and `payments` (`user_id`, `contract_id`, a unique `transaction_id` UUID, `status` = `pending|completed|failed`, `value`).
- In the template's `plans` migration, the `active` column was renamed to `is_active`.
- **Models** `Contract` and `Payment` with relationships. `Plan` and `User` gain relationships, `$fillable` and casts, and `User` gets a `current_contract` accessor for the active contract.
- **`PaymentObserver`** reacts when a payment's `status` changes:
  - `completed`: the contract becomes active and the user's other contracts are deactivated. `started_at` is set if empty, and `ended_at` is set to one month ahead or extended by one month.
  - Any other status deactivates the contract.
- **Plan hiring with credit** (`PlanController@hire`):
  - A new inactive contract is created at the plan's price.
  - If the user has an active contract, its unused days become a credit of `contract value * remaining days / days in the contract's first month`.
  - Payments are then created in a loop. Each one is `max(price - remaining credit, 0)`, and the credit drops by the plan price after each payment.
  - So a credit larger than the new price produces several payments. Zero-value payments are completed straight away, which activates the contract through the observer.
- **Endpoints**, listed in the next section.
- **Laravel Sail setup**: `api/docker-compose.yml` (PHP 8.2 runtime, PostgreSQL 15, Redis) and `api/Makefile`.

## API

There is no authentication, as the template states. Every endpoint works on the user with id `1`.

| Method | Path | Origin | Description |
| --- | --- | --- | --- |
| GET | `/api/plans` | template | All plans |
| GET | `/api/user` | template | The "logged-in" user |
| GET | `/api/user/contracts` | author | The user's contracts, with plan and payments |
| GET | `/api/user/contracts/current` | author | The active contract, with plan and payments, or `null` |
| POST | `/api/user/plans/{plan}/hire` | author | Creates the contract and pending payment(s); returns the last payment |
| POST | `/api/payments/{transaction_id}/process` | author | Body `status` = `pending\|completed\|failed`; simulates the payment confirmation |

## Tech stack

- **API**: PHP `^8.0.2`, Laravel 9.52, Sanctum 3.3 (installed but unused), PHPUnit 9.6, PostgreSQL. Versions are from `api/composer.lock`.
- **Front-end** (template only): React 18, react-router-dom 6, Vite 4, TypeScript 5, Tailwind CSS 3.

## Running locally

**Option A, the template's own steps:**

```bash
(cd db && docker compose up -d)          # PostgreSQL on 5432, database "desafio"
cd api
cp .env.example .env
composer install
php artisan key:generate
php artisan migrate
php artisan db:seed
php artisan serve                        # http://127.0.0.1:8000/api
```

**Option B, Laravel Sail** (the author's setup, through `api/Makefile`):

```bash
cd api
cp .env.example .env                     # set DB_HOST=pgsql for the Sail network
make install                             # composer install
make start                               # sail up -d (PostgreSQL forwarded to 54322)
./vendor/bin/sail artisan key:generate
make seed                                # migrate --seed
```

**Front-end** (it shows only the template's placeholder page):

```bash
cd front && npm install && npm run dev
```

## Tests

```bash
cd api && make test                      # or: php artisan test
```

Only the template's Laravel example tests exist. The author added no tests. `phpunit.xml` points at a `testing` database, which Sail's PostgreSQL init script creates.

## Project structure

```
api/app/Http/Controllers/   PlanController (hire), ContractController, PaymentController, UserController
api/app/Models/             Plan, User, Contract, Payment
api/app/Observers/          PaymentObserver
api/database/migrations/    plans, users (template); contracts, payments (author)
api/routes/api.php          API routes
db/docker-compose.yml       PostgreSQL (template)
front/                      React scaffold (template, unchanged)
```

## Notes

- Only the back end was built. The front-end screens that the statement asks for are not implemented: the plan list, the Pix payment page, the current plan and the history.
- The credit is based on the old contract's `value` (the plan price), not on what the user actually paid for it.
- `hire` does not stop the user from hiring their current plan again. Hiring without paying leaves extra inactive contracts with pending payments.
- The `contracts` and `payments` foreign keys use `onDelete('set null')` on columns that are not nullable.
- The Makefile's `sh:mysql` target points at a `mysql` service that does not exist in this compose file.
- `api/artisan` lost its executable bit, so use `php artisan`.
- `vendor/` and `node_modules/` are not committed.
