# React and Laravel Implementation Guide

This guide translates the product specification into the required challenge stack. The business behavior in `spec-python/` and `spec-.net/` remains the source of truth; their Python and .NET implementation choices are reference material only.

## Stack mapping

| Specification concept | Challenge implementation |
|---|---|
| Python/.NET HTTP backend | PHP 8.3+ with Laravel 13 |
| Domain and application rules | Small PHP domain objects and application actions/services, independent of controllers and Eloquent |
| Repositories and persistence adapters | Laravel Eloquent repositories behind application interfaces |
| Alembic or EF Core migrations | Laravel migrations; PostgreSQL owns every invariant marked as database-enforced |
| FastAPI/ASP.NET HTTP contract | Laravel routes, Form Requests, API Resources, and a single exception-to-error-response mapper |
| Backend integration tests | Laravel feature tests against PostgreSQL; focused domain tests for business rules |
| Operations client | React + TypeScript, calling only `/api/v1` |
| Demo-data seeder | A separate CLI utility that calls the API, never SQL directly |
| Deployment and local services | Docker Compose with PostgreSQL, API, React app, and static page |

## API structure

Keep the Laravel HTTP layer thin:

```text
app/
  Domain/             # Book, Loan, LoanItem, Member, User and value objects
  Application/        # use cases, policies, repository interfaces, clock/token/hash ports
  Infrastructure/     # Eloquent repositories, JWT, cover storage, system clock
  Http/Controllers/Api/V1/
  Http/Requests/Api/V1/
  Http/Resources/Api/V1/
```

Controllers validate through Form Requests, call one application action, and return a resource or the shared error shape. Keep role checks in explicit authorization policies or middleware. Do not expose Eloquent models or password hashes directly in API responses.

## Persistence and concurrency

- Use PostgreSQL 16 and Laravel migrations for the six tables, indexes, foreign keys, constraints, and the five fixed categories.
- Preserve the physical schema in `data-model.md`; do not add defaults or audit columns without updating the specification first.
- Record a loan and decrement every book's available count in one database transaction. Use row-level locks or an atomic conditional decrement so concurrent requests cannot loan the last copy twice.
- A failed item in a multi-book loan rolls back every decrement and insert.
- Calculate loan status and days late at read time. Snapshot title, ISBN, and category name on each loan item.

## Authentication and security

- The API contract requires a signed JWT with `sub`, `role`, and an eight-hour expiry. A framework session or opaque personal-access token is not a substitute for that contract.
- Hash passwords using a password-hashing library supported by PHP; never log credentials or tokens.
- Serve credential and token endpoints over HTTPS.
- Read secrets and deployment configuration from the environment; commit only safe example values.
- Validate uploaded image content and size, store the binary outside the database, and persist only an opaque `cover_key`.

## React application

Keep API access in a small typed client. Build screens from the acceptance criteria: sign-in, book catalog/search, member management, loan creation, item return, overdue list, and admin report. Represent loading, empty, success, and error states explicitly; do not duplicate business rules that belong to the API.

All code, documentation, interface text, validation messages, and API error messages for this challenge are in English. Keep field names, routes, codes, and accepted status values exactly as specified.

## Six-repository boundaries

| Repository | Owns |
|---|---|
| `docs` | Product requirements, API contract, data model, and this stack translation |
| `api` | Laravel application, migrations, API, and automated checks |
| `app` | React operations interface |
| `page` | Static public presentation; never calls the API |
| `infra` | Docker Compose, PostgreSQL, networks, and volumes |
| `tool` | Repeatable demo-data creation through the API |

## Traceable delivery

Implement one reviewable acceptance-criteria slice at a time. Each commit uses the next `hu-main-NN` identifier, states the change in English, and includes the relevant evidence. Update this guide and the source specification together whenever a business or contract decision changes.
