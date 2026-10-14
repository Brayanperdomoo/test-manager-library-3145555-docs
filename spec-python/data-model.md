# Data Model — Manager Library

**The single source of truth for the data model.** Implementers should not need to inspect code or connect to the database to learn what exists, its type, or its constraints.

- Reference engine: PostgreSQL 16. Schema: `library`. Server timezone: UTC.
- Governed by [`constitution.md`](constitution.md) and [`spec.md`](spec.md). If anything conflicts with this document, this document takes precedence.

Each rule has one of **three labels**: **database** (enforced by the database), **domain-only** (enforced by application code only), or **pending (T-xx)** (implemented by the task in [`tasks.md`](tasks.md)).

---

## 0. Naming conventions

| Element | Convention | Example |
|---|---|---|
| Table | `snake_case`, English, **singular** | `loan_item` |
| Column | `snake_case`, English, singular, ASCII | `due_at` |
| Hand-written constraint | `ck_{table}_{rule}` | `ck_book_available_range` |
| Schema | `library` | `library.loan` |

In handwritten SQL, `user` is written as `library."user"`; the ORM quotes it automatically.

## 1. Glossary

| Term | Definition | Stored in |
|---|---|---|
| **Book** | Catalog title with author, ISBN, category, copies, and optional cover | `book` |
| **Category** | One of five fixed categories, seeded and not maintained | `category` |
| **Copy** | A physical copy of a book. Counted, not individually identified | `book.total_copies`, `book.available_copies` |
| **Member** | Person who borrows books. Does not sign in | `member` |
| **Loan** | Completed event: who borrowed, who recorded it, and when | `loan` |
| **Loan item** | A book in a loan, with its due date and return | `loan_item` |
| **Loan status** | `open`, `overdue`, or `closed`. **Calculated, not stored** | no column |
| **User** | Internal operator with `admin` or `librarian` role | `user` |
| **Snapshot** | Value copied at loan time; does not follow later catalog changes | `loan_item.book_title`, `isbn`, `category_name` |

## 2. Entities and relationships

```mermaid
erDiagram
    category ||--o{ book      : "classifies"
    member   ||--o{ loan      : "requests"
    user     ||--o{ loan      : "records"
    loan     ||--|{ loan_item : "contains"
    book     ||--o{ loan_item : "borrowed in (FK RESTRICT)"
```

Six entities, six tables. No report, fines, or audit table.

## 3. Physical model

No column has a `DEFAULT`: values are set by the domain.

**`category`** — read-only seed data.

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK. Literal identifiers (see §9) |
| `name` | `varchar(120)` | no | Unique |

**`book`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `title` | `varchar(250)` | no | Trimmed by the domain |
| `author` | `varchar(200)` | no | |
| `isbn` | `char(13)` | no | Digits only. Unique among active books (partial index) |
| `category_id` | `uuid` | no | Restrictive FK to `category` |
| `total_copies` | `integer` | no | ≥ 1 |
| `available_copies` | `integer` | no | Between 0 and `total_copies` |
| `cover_key` | `varchar(512)` | **yes** | Opaque key; never a path or bytes; never an empty string |
| `deleted_at` | `timestamptz` | yes | Soft delete. Null = active |

**`member`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `full_name` | `varchar(200)` | no | |
| `document_number` | `varchar(30)` | no | Unique |

**`loan`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `member_id` | `uuid` | no | Restrictive FK to `member` |
| `loaned_by_user_id` | `uuid` | no | Restrictive FK to `user` |
| `loaned_at` | `timestamptz` | no | Loan instant |

**`loan_item`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `loan_id` | `uuid` | no | FK with `ON DELETE CASCADE` |
| `book_id` | `uuid` | no | Restrictive FK to `book` |
| `book_title` | `varchar(250)` | no | Snapshot; same width as `book.title` |
| `isbn` | `char(13)` | no | Snapshot |
| `category_name` | `varchar(120)` | no | Snapshot; intentionally **no FK** |
| `due_at` | `timestamptz` | no | `loaned_at` + 14 days, calculated by the domain |
| `returned_at` | `timestamptz` | **yes** | Null until returned |

**`user`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `username` | `varchar(120)` | no | Unique, lowercase, trimmed |
| `password_hash` | `varchar(512)` | no | Never indexed |
| `role` | `varchar(40)` | no | `admin` or `librarian` |

Cross-cutting rules: **all timestamps are `timestamptz`**; **there are no audit columns** (`created_at`/`updated_at`) because no requirement calls for them.

## 4. Constraints and indexes: where each rule lives

| Rule | Object | Label |
|---|---|---|
| Primary keys on all six tables | `PK_*` | **database** |
| `category.name` unique | unique index | **database** |
| `member.document_number` unique | unique index | **database** |
| `user.username` unique | unique index | **database** |
| `isbn` unique among active books | partial unique index `WHERE deleted_at IS NULL` | **database** |
| `book.category_id` → `category.id` RESTRICT | FK | **database** |
| `loan.member_id` → `member.id` RESTRICT | FK | **database** |
| `loan.loaned_by_user_id` → `user.id` RESTRICT | FK | **database** |
| `loan_item.loan_id` → `loan.id` CASCADE | FK | **database** |
| `loan_item.book_id` → `book.id` RESTRICT | FK | **database** |
| `0 <= available_copies <= total_copies` | `ck_book_available_range` | **database** — final concurrency safeguard |
| `total_copies >= 1` | `ck_book_total_positive` | **pending (T-18)** · domain only today |
| Unique `(loan_id, book_id)` | unique index | **database** |
| `role` in (`admin`,`librarian`) | `ck_user_role` | **pending (T-18)** · domain only today |
| Maximum 3 unreturned items per member | — | **domain-only** (not expressible as a `CHECK`) |
| Member with overdue items cannot borrow | — | **domain-only** |
| Deducting a copy and creating an item are one operation | — | **domain-only** (one transaction) |
| A loan has 1 to 3 items | — | **domain-only** |
| Loans and returns are immutable except for `returned_at` | — | **domain-only** (no other operation exists) |

Access indexes: partial `book (category_id, title)` for active books; `loan (loaned_at)`; `loan_item (book_id)`; `loan_item (due_at) WHERE returned_at IS NULL` for overdue items.

## 5. Foreign-key policy

| # | FK | `ON DELETE` | Reason |
|---|---|---|---|
| FK-1 | `book.category_id` | RESTRICT | A category with books cannot be deleted |
| FK-2 | `loan_item.loan_id` | CASCADE | Composition: an item cannot exist without its loan (never triggered: loans are not deleted) |
| FK-3 | `loan_item.book_id` | RESTRICT | Last-resort safeguard: manual deletion cannot orphan history |
| FK-4 | `loan.member_id` | RESTRICT | The member on a loan cannot disappear |
| FK-5 | `loan.loaned_by_user_id` | RESTRICT | Authorship is an accounting record |

`ON UPDATE NO ACTION` for all: application UUIDs do not change.

## 6. Access patterns

| # | Pattern | Frequency |
|---|---|---|
| Q1 | Search books (text, category, active only, title sort, pagination) | High |
| Q2 | Book by ID | High |
| Q3 | Books by a batch of IDs (before lending) | High, contention point |
| Q4 | List categories | High |
| Q5 | Loan with its items | Medium |
| Q6 | Loans by range, status, and member | High |
| Q7 | A member's unreturned items (count and overdue checks) | High, on every loan |
| Q8 | All overdue items | Medium |
| Q9 | Most-borrowed report by range | High, most expensive |
| Q10 | User by username | High, on every sign-in |

## 7. Privacy and retention

| Table | Attribute | Classification | Handling |
|---|---|---|---|
| `user` | `username` | Personal data | Restricted access |
| `user` | `password_hash` | **Secret** | Never in logs or responses; never indexed |
| `member` | `full_name`, `document_number` | **Personal data** | Admin and librarian only; **never** in reports or public endpoints |
| `loan`, `loan_item` | all | Business data; `member_id` is indirectly personal data | Retain indefinitely |
| `book`, `category` | all | Public | Soft-delete / no deletion |

Reports aggregate by book: **they do not expose members or librarians**.

**Retention:** loans and items are never deleted; books are only soft-deleted; the cover binary is the **only** data physically deleted. Required order: first clear `cover_key` and commit the transaction; then delete the binary.

## 8. Auditing

The system has no `created_at`/`updated_at`. Business instants already have columns: `loan.loaned_at`, `loan_item.returned_at`, and `book.deleted_at`. If someone asks, “Who changed this title?”, the decision goes back to the product owner.

## 9. Seed strategy

The **five categories** are inserted by the initial migration with fixed identifiers:

| `id` | `name` |
|---|---|
| `11111111-1111-4111-8111-111111111111` | Novel |
| `22222222-2222-4222-8222-222222222222` | Science |
| `33333333-3333-4333-8333-333333333333` | Technology |
| `44444444-4444-4444-8444-444444444444` | History |
| `55555555-5555-4555-8555-555555555555` | Children's |

The database does **not** seed the initial administrator: application startup creates it from environment credentials because only the password-hashing port can produce the hash. The demo data seeder (`tool`) uses the **API**, never direct SQL.

## 10. How to verify this document

```sql
-- Must return the columns in §3, with no DEFAULT values
SELECT table_name, ordinal_position, column_name, data_type, is_nullable, column_default
FROM information_schema.columns WHERE table_schema = 'library' ORDER BY 1, 2;

-- Constraints: PK, FK, and CHECK
SELECT c.conrelid::regclass, c.conname, c.contype, pg_get_constraintdef(c.oid)
FROM pg_constraint c JOIN pg_namespace n ON n.oid = c.connamespace
WHERE n.nspname = 'library' ORDER BY 1, 3, 2;

-- Indexes
SELECT tablename, indexname, indexdef FROM pg_indexes WHERE schemaname = 'library' ORDER BY 1, 2;
```

If a rule labeled **database** is missing, someone removed it. If a **domain-only** rule appears in the database, it has been implemented there and must be relabeled. The database wins: the document is wrong and must be corrected.
