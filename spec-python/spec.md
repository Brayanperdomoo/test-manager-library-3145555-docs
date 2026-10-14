# Specification — Manager Library

**What is being built and why.** A system for a small library to manage its book catalog, members, and loans, and to see which books are borrowed most often.

This specification is governed by [`constitution.md`](constitution.md). The data model is defined only in [`data-model.md`](data-model.md), and the HTTP contract only in [`api-contract.md`](api-contract.md).

---

## 1. Actors

| Actor | Responsibilities | Authentication |
|---|---|---|
| **Administrator** (`admin`) | Maintains the book catalog and members; creates librarian accounts; views all reports | Username and password → token |
| **Librarian** (`librarian`) | Records loans and returns; searches the catalog and members; views overdue items | Username and password → token |
| **Member** | Person who borrows books. **Does not sign in**: this is a record, not a user | — |

The `admin` role is provisioned by deployment from environment variables. **It cannot be granted at runtime.**

## 2. Scope

**In scope:** book catalog with optional cover, five fixed categories, members, loans with multiple items, item-by-item returns, overdue items, most-borrowed report, role-based sign-in.

**Out of scope and not subject to reopening:** fines and payments, reservations, renewals, email or SMS, bulk import, inter-branch loans, and book attributes beyond those in §3 (no publisher, year, or language).

## 3. User stories and acceptance criteria

### HU-01 · Sign in
As a user, I want to authenticate so I can operate according to my role.
- **CA-01.1** Valid credentials return a token valid for 8 hours.
- **CA-01.2** Invalid credentials return 401 with a message that does **not** reveal whether the username or password was incorrect.
- **CA-01.3** Every endpoint except sign-in and `/health` requires a token. No token: 401. Insufficient role: 403.

### HU-02 · Create librarian accounts (admin)
- **CA-02.1** The admin creates a user with a username, password, and `librarian` role.
- **CA-02.2** The username is stored trimmed and lowercased; it is unique. Duplicate: 409.
- **CA-02.3** Requesting the `admin` role through the API returns 422. The password is never returned or logged.

### HU-03 · Manage books (admin)
- **CA-03.1** Create a book with title, author, ISBN, category, and total copies. Available copies start equal to the total.
- **CA-03.2** The ISBN has 13 digits and is unique among active books. Duplicate: 409.
- **CA-03.3** Total copies ≥ 1. Changing the total adjusts available copies by the same difference and **never** makes available copies < 0 (422).
- **CA-03.4** Uploading a cover (JPG/PNG, max. 2 MB) stores an **opaque key**; without a cover, the field is `null`, never an empty string.
- **CA-03.5** Deleting a book is a **soft delete**: it disappears from the catalog but remains in past loans. A book with open loans cannot be soft-deleted (409).

### HU-04 · Search books
- **CA-04.1** Partial text search by title or author, category filter, active books only, title sort, and pagination (`page`, `size`, max. 50).
- **CA-04.2** The response includes the total number of results.
- **CA-04.3** Looking up a nonexistent or soft-deleted book by ID returns 404.

### HU-05 · Manage members
- **CA-05.1** Admins and librarians can create members with a name and document number. The document number is unique: duplicate returns 409.
- **CA-05.2** Members can be listed and searched by name or document number, with pagination.
- **CA-05.3** Members cannot be deleted.

### HU-06 · Record a loan
- **CA-06.1** A loan belongs to **one** member, is recorded by the authenticated user, and contains 1 to 3 items.
- **CA-06.2** The same book cannot appear more than once in a loan (422).
- **CA-06.3** Each item deducts **one** available copy in the same operation. If none are available: 409 and **nothing is recorded**.
- **CA-06.4** A member cannot have more than **3 unreturned books** in total (422 with the current count).
- **CA-06.5** A member with any **overdue** item cannot borrow another book (409).
- **CA-06.6** The due date is `loaned_at + 14 days`. The item snapshots the title, ISBN, and category name **at that moment**.
- **CA-06.7** A recorded loan cannot be edited or deleted.

### HU-07 · Record a return
- **CA-07.1** Items are returned **one at a time**. Set `returned_at` and add one to available copies.
- **CA-07.2** Returning an already returned item returns 409. **Never** add the copy twice.
- **CA-07.3** Available copies must never exceed total copies (guaranteed by the database).
- **CA-07.4** When all items are returned, the loan is **closed**.

### HU-08 · View and list loans
- **CA-08.1** Viewing a loan returns its items and calculated status (`open`, `overdue`, `closed`).
- **CA-08.2** List loans by date range, status, and member; sort by date descending; paginate.
- **CA-08.3** Status is **calculated**, not stored.

### HU-09 · Overdue items
- **CA-09.1** List unreturned items whose `due_at` is earlier than now, including book, member, and days late.
- **CA-09.2** Sort by most days late first. Admin and librarian only.

### HU-10 · Most-borrowed books report
- **CA-10.1** For a date range, return one row per **book and snapshotted category name**, with the number of times borrowed, sorted from highest to lowest.
- **CA-10.2** End date before start date: 422. A range with no loans returns an empty list, not an error.
- **CA-10.3** The report is **not** broken down by librarian or member.
- **CA-10.4** A report for a closed period **does not change** if the book title later changes or the book is reassigned to another category. The report uses the title and category name snapshotted on each loan item; categories are fixed and cannot be renamed.
- **CA-10.5** Admin only.

## 4. Business rules

| # | Rule |
|---|---|
| BR-1 | Available copies are always between 0 and total copies. |
| BR-2 | Loan period: 14 days. Maximum 3 unreturned books per member. |
| BR-3 | A member with overdue items cannot borrow more until they return them. |
| BR-4 | Snapshotted item data (title, ISBN, category) does not follow later catalog changes. |
| BR-5 | Loans and returns are never deleted. Books are only soft-deleted. |
| BR-6 | Timestamps are UTC instants. User-facing messages, code, and column names are in English. |

## 5. Non-functional requirements

| # | Requirement | Measurement |
|---|---|---|
| NFR-1 | Book search responds in under 300 ms with 5,000 books | Load test using the `tool` seeder |
| NFR-2 | If two librarians loan the **last available copy** at the same time, one succeeds and the other receives 409 | Concurrent test |
| NFR-3 | Passwords are transmitted only over HTTPS, are never stored in plaintext, and never appear in logs | Review TLS configuration, logs, and responses |
| NFR-4 | Every HTTP error uses the same response body (see contract) | Contract test |
| NFR-5 | The system starts with one command and the database starts empty | `infra` README |
