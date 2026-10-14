# API Contract — Manager Library

The HTTP contract is **language-independent**: an implementation in React + Laravel must follow it just as an implementation in Python or .NET would. What each operation does is defined in [`spec.md`](spec.md); what is stored is defined in [`data-model.md`](data-model.md).

## 1. Conventions

| Topic | Rule |
|---|---|
| Base path | `/api/v1` |
| Format | JSON, UTF-8. Field names use `snake_case` |
| Identifiers | UUID v4 as text |
| Dates | ISO 8601 in UTC, with `Z` (`2026-10-14T14:15:00Z`) |
| Money | None. The system does not handle monetary amounts |
| Authentication | `Authorization: Bearer <token>` on every endpoint except `POST /auth/login` and `GET /health` |
| Pagination | `?page=1&size=20`. `size` max. 50. Response: `{ "items": [...], "page": 1, "size": 20, "total": 134 }` |
| Language | Error messages are in Spanish; fields and codes are in English |

### Error body (same shape for every error)

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`details` may be empty. It **never** includes stack traces, SQL, server paths, or the password.

### Status codes

| Status | When |
|---|---|
| 200 / 201 / 204 | Success / created / no content |
| 401 | Missing, invalid, or expired token; invalid credentials |
| 403 | Insufficient role |
| 404 | Does not exist (or is soft-deleted) |
| 409 | State conflict: duplicate, no copies available, already returned, member has overdue items, book has open loans |
| 422 | Validation: invalid field, broken input rule, inverted date range |

## 2. Business error codes

| `code` | Status | Source |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | CA-01.2 |
| `FORBIDDEN_ROLE` | 403 | CA-01.3 |
| `NOT_FOUND` | 404 | Any resource |
| `USERNAME_TAKEN` | 409 | CA-02.2 |
| `ISBN_TAKEN` | 409 | CA-03.2 |
| `DOCUMENT_TAKEN` | 409 | CA-05.1 |
| `BOOK_HAS_OPEN_LOANS` | 409 | CA-03.5 |
| `BOOK_NOT_AVAILABLE` | 409 | CA-06.3 |
| `MEMBER_HAS_OVERDUE` | 409 | CA-06.5 |
| `ITEM_ALREADY_RETURNED` | 409 | CA-07.2 |
| `MEMBER_LIMIT_EXCEEDED` | 422 | CA-06.4 |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | CA-06.2 |
| `VALIDATION_ERROR` | 422 | Any invalid field |
| `INVALID_DATE_RANGE` | 422 | CA-10.2 |

## 3. Endpoints

### 3.1 Authentication and health

| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/auth/login` | public | Returns a token |
| GET | `/health` | public | `{ "status": "ok" }` |

`POST /auth/login`

```json
// request
{ "username": "ana", "password": "********" }
// 200
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 28800, "role": "librarian" }
```

### 3.2 Users

| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/users` | admin | Creates a librarian |

```json
// request (role only accepts "librarian")
{ "username": "Ana ", "password": "********", "role": "librarian" }
// 201 (no password or password_hash)
{ "id": "…", "username": "ana", "role": "librarian" }
```

### 3.3 Categories

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/categories` | authenticated | The five categories, sorted by name |

### 3.4 Books

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/books` | authenticated | Search: `q`, `category_id`, `page`, `size` |
| GET | `/books/{id}` | authenticated | Details |
| POST | `/books` | admin | Creates a book |
| PUT | `/books/{id}` | admin | Replaces title, author, category, and total |
| DELETE | `/books/{id}` | admin | Soft-deletes a book |
| PUT | `/books/{id}/cover` | admin | Uploads cover (`multipart/form-data`, field `file`) |
| GET | `/books/{id}/cover` | authenticated | Returns the cover file; 404 if none exists |
| DELETE | `/books/{id}/cover` | admin | Removes the cover |

Book response:

```json
{
  "id": "…", "title": "Cien años de soledad", "author": "Gabriel García Márquez",
  "isbn": "9780307474728", "category": { "id": "…", "name": "Novela" },
  "total_copies": 4, "available_copies": 3, "cover_key": null
}
```

`POST /books` request fields: `title`, `author`, `isbn`, `category_id`, `total_copies`. The server calculates available copies.
`cover_key` is an opaque key: clients retrieve the image using `GET /books/{id}/cover`, which returns the binary file or 404.

### 3.5 Members

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/members` | admin, librarian | Lists and searches: `q`, `page`, `size` |
| GET | `/members/{id}` | admin, librarian | Details with `active_loans` (count of unreturned items) |
| POST | `/members` | admin, librarian | Creates with `full_name`, `document_number` |

### 3.6 Loans

| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/loans` | admin, librarian | Records a loan |
| GET | `/loans` | admin, librarian | Lists by `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | `/loans/{id}` | admin, librarian | Details with items and status |
| POST | `/loans/{loan_id}/items/{item_id}/return` | admin, librarian | Returns one item |
| GET | `/loans/overdue` | admin, librarian | Lists overdue items |

`POST /loans`

```json
// request (1 to 3 items, no duplicate books)
{ "member_id": "…", "items": [ { "book_id": "…" }, { "book_id": "…" } ] }
// 201
{
  "id": "…", "member_id": "…", "loaned_at": "2026-10-14T14:15:00Z", "status": "open",
  "items": [
    { "id": "…", "book_id": "…", "book_title": "Cien años de soledad", "isbn": "9780307474728",
      "category_name": "Novela", "due_at": "2026-10-28T14:15:00Z", "returned_at": null }
  ]
}
```

`status` is calculated: `open` (unreturned items, none overdue), `overdue` (at least one unreturned item is overdue), or `closed` (all items returned).

`POST /loans/{loan_id}/items/{item_id}/return` → 200 with the item (`returned_at` populated).

`GET /loans/overdue` → `items`: `{ "loan_id", "item_id", "book_title", "member_name", "due_at", "days_late" }`. This operational list includes the member's name; **the §3.7 report does not**.

### 3.7 Report

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/reports/most-borrowed` | admin | Required `from` and `to` dates, both inclusive; filters by `loan.loaned_at` in UTC |

```json
{
  "from": "2026-10-01", "to": "2026-10-31",
  "rows": [
    { "book_id": "…", "book_title": "Cien años de soledad", "category_name": "Novela", "times_borrowed": 12 }
  ]
}
```

One row per snapshotted `book_id + book_title + category_name`. Sort by `times_borrowed` descending, then title. No members or librarians.

`from` includes 00:00:00 UTC on that date. `to` includes up to, but not including, 00:00:00 UTC on the following date. This includes all loans on the `to` date.

## 4. Security

| Aspect | Rule |
|---|---|
| Token | Signed JWT, 8 hours, with `sub` and `role`. No personal data inside |
| Transport | Endpoints that receive credentials or a token are available only over HTTPS |
| Passwords | Only the hash is stored; the hash is never included in a response |
| Authorization | Checked on every endpoint by role; a 403 does not reveal whether a resource exists |
| CORS | Only the frontend origin configured through the environment |
| Cover upload | Only `image/jpeg` and `image/png`, max. 2 MB; validate type by content, not extension |
