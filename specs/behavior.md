# Easy Exchange — Behavior

## Behavioral specification (user flows)

**Product:** Easy Exchange (book exchange)  
**Status:** Planning  
**Related:** `requirements.md` (what), `architecture.md` (where)

This document describes **step-by-step** v1 flows. Implementation and automated tests should follow these sequences, including alternate and error paths.

Actors in examples:

- **Maya** — listing owner (has a calculus textbook to exchange).
- **Jordan** — requester (wants Maya’s book; may offer a novel in return).

---

## Flow A — List a book

### A.1 Happy path (authenticated)

**Precondition:** Maya has a valid account and is signed out or signed in. The catalog may already contain other books.

| Step | Actor | System response |
| --- | --- | --- |
| 1 | Maya opens the app home/catalog. | Public `Available` listings render. Header shows **Sign in** if she is a guest. |
| 2 | If signed out, Maya chooses **Sign in**, enters email and password, submits. | Session is created. Header shows her display name, **List a book**, **My listings**, **Requests**. |
| 3 | Maya chooses **List a book**. | `ListingForm` appears with fields: title, author, condition, description, optional cover, optional course/subject tag, trade notes. |
| 4 | Maya fills: title `Calculus: Early Transcendentals`, author `Stewart`, condition `Good`, description (highlights, no missing pages), tag `MATH 101`, trade notes `Looking for any linear algebra text or pickup on campus`. She optionally attaches a JPEG under 5 MB. | Client-side checks: required title/author/condition; file type/size if present. |
| 5 | Maya submits. | Server validates (Zod), requires a session, stores the listing with status **`Available`**, stores image if provided, redirects to the new listing detail (or My Listings). Success message: listing is live. |
| 6 | Maya opens **My listings**. | The calculus book appears with status `Available`. |
| 7 | A guest or Jordan opens Search and queries `calculus` or filter tag `MATH 101`. | Maya’s listing appears in results (publicly searchable). |

**Postcondition:** One `Listing` row owned by Maya, status `Available`, visible in public search. No trade requests exist yet.

### A.2 Alternate — unauthenticated create

| Step | Actor | System response |
| --- | --- | --- |
| 1 | Guest clicks **List a book**. | Redirect to **Sign in** with return URL `/books/new`. |
| 2 | Guest registers or signs in. | After success, redirect back to the listing form (empty). |
| 3 | Continue from A.1 step 4. | Same as happy path. |

### A.3 Validation and errors (listing)

| Situation | System behavior |
| --- | --- |
| Title or author blank | Form does not create a listing; inline error on the field. HTTP 422 if API called directly. |
| Condition not in `New` / `Like New` / `Good` / `Acceptable` | Rejected. |
| Image 6 MB or `.gif` | Rejected with message about JPEG/PNG/WebP and 5 MB max. Listing is not created (atomic: no orphan listing without intended image if upload is part of the same submit; if upload is a separate step, listing may exist without image). |
| Not signed in on POST | 401; redirect to sign-in. |
| Database down | Generic error; no stack trace. Listing not created. |

### A.4 Owner follow-ups (not the core “list” path, but related)

- **Edit:** Maya changes description while status is `Available`. Public detail updates.
- **Withdraw:** Maya withdraws. Listing status `Withdrawn`; it disappears from public search. If Jordan had a `Pending` request, that request becomes `Cancelled` and Jordan gets a notification.

---

## Flow B — Complete a trade request

“Complete” means: Jordan requests Maya’s book, Maya accepts, they hand off offline, Maya marks the trade completed.

### B.1 Happy path

**Precondition:** Flow A completed. Jordan has an account and at least one of: (a) an `Available` listing to offer, or (b) willingness to request pickup-only with a message. Maya’s calculus listing is `Available`. Jordan is not the owner.

| Step | Actor | System response |
| --- | --- | --- |
| 1 | Jordan searches `calculus` or browses the catalog. | Maya’s listing appears (`Available`). |
| 2 | Jordan opens the listing detail. | Sees title, author, condition, trade notes, Maya’s display name. Primary action: **Request trade**. Jordan does **not** see Edit/Withdraw. |
| 3 | Jordan clicks **Request trade**. | `RequestForm`: optional offered listing(s) from Jordan’s `Available` books; required message; optional handoff note (e.g. `Library lobby, Friday 3pm`). |
| 4 | Jordan selects his `Linear Algebra` listing (optional), writes a message, submits. | Server checks: session, Jordan ≠ owner, listing still `Available`, no other open request from Jordan on this listing. Creates `TradeRequest` status **`Pending`**. Snapshots listing title/author. Creates notification for Maya. Redirects to request detail. Listing remains **`Available`** (other people can still request until accept). |
| 5 | Maya sees unread count in the header and opens **Requests** (received). | New row: Jordan, calculus book, `Pending`. |
| 6 | Maya opens the request. | Sees Jordan’s message, offered book (if any), handoff note. Actions: **Accept**, **Decline**. |
| 7 | Maya clicks **Accept**. | **Transaction:** this request → `Accepted`; any other `Pending` requests on the same listing → `Declined` (those requesters notified); listing → **`Pending Trade`**. Jordan is notified of acceptance. Listing drops out of default public catalog (only `Available` in public search). |
| 8 | Maya and Jordan meet using the handoff note (outside the app). | No system change. |
| 9 | Maya opens the accepted request and clicks **Mark completed** (v1: owner confirmation). | Request → **`Completed`**. Listing → **`Exchanged`**. Jordan notified. Listing stays on Maya’s My Listings as historical; not in public search. Jordan’s offered book, if any, is **not** auto-exchanged in v1 unless it was the target of a separate listing flow (v1 completes the **target** listing only). |

**Postcondition:** Target listing `Exchanged`. Winning request `Completed`. Sibling requests `Declined`. Catalog no longer shows the calculus book.

### B.2 Alternate — decline

After B.1 step 6, Maya clicks **Decline**.

- Request → `Declined`. Jordan notified.
- Listing stays `Available`.
- Other pending requests unchanged.

### B.3 Alternate — requester cancels before accept

After B.1 step 4, Jordan opens the request and **Cancels**.

- Request → `Cancelled`. Maya notified.
- Listing stays `Available`.

### B.4 Alternate — cancel after accept

After B.1 step 7, either party **Cancels**.

- Request → `Cancelled`.
- Listing returns to `Available` (no remaining accepted request).
- Book is searchable again.
- The other party is notified.

### B.5 Alternate — competing requests

**Precondition:** Priya also sent a `Pending` request on Maya’s listing.

When Maya accepts Jordan’s request:

- Jordan: `Accepted`
- Priya: `Declined` + notification
- Listing: `Pending Trade`
- Maya cannot accept Priya afterward (request already terminal).

### B.6 Error and forbidden paths

| Situation | System behavior |
| --- | --- |
| Jordan requests his own listing | CTA hidden; POST returns 403. |
| Guest clicks Request trade | Redirect to sign-in with return URL to listing. |
| Listing is `Pending Trade` or `Exchanged` or `Withdrawn` | Cannot create a new request; 409 or 404 for withdrawn. |
| Jordan already has `Pending` or `Accepted` on this listing | 409; message to open the existing request. |
| Priya tries to accept Jordan’s request | 403. |
| Jordan tries to mark complete (v1) | 403; only owner completes. |
| Maya accepts a request that is already `Declined` | 409 illegal transition. |
| Two accepts in a race | Transaction + unique business rule: at most one `Accepted` per listing at a time; loser gets error/409. |

---

## Flow C — Sign-in gate (shared)

Used by List and Trade when the session is missing.

1. User hits a protected URL or action.  
2. System redirects to `/login?callbackUrl=...`.  
3. Successful login restores the callback URL.  
4. Failed login (wrong password) shows a generic “Invalid email or password” (do not reveal whether the email exists).

---

## Status cheat sheet (for QA)

### Listing

```text
Available  --list-->  (start)
Available  --owner withdraw-->  Withdrawn
Available  --owner accepts a request-->  Pending Trade
Pending Trade  --complete-->  Exchanged
Pending Trade  --cancel accepted request-->  Available
```

### Trade request

```text
Pending  --accept-->  Accepted  --complete-->  Completed
Pending  --decline-->  Declined
Pending  --cancel-->  Cancelled
Accepted --cancel-->  Cancelled
```

---

## Test outline (implementation phase)

Map automated tests to this file:

1. **List book:** authenticated create → appears in My Listings and search.  
2. **List book:** guest redirected.  
3. **Trade:** create pending request; owner accept → listing `Pending Trade`; sibling declined.  
4. **Trade:** complete → listing `Exchanged`; gone from public search.  
5. **Trade:** decline leaves listing `Available`.  
6. **Authz:** non-owner cannot accept or edit.

Manual QA: run Flow A then Flow B once on desktop and once on a 375px-wide viewport.
