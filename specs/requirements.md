# Easy Exchange — Book Exchange Platform

## Requirements Specification

**Product:** Easy Exchange (book-focused peer-to-peer exchange)  
**Document type:** Functional and non-functional requirements  
**Status:** Planning (no implementation yet)

---

## 1. Product overview

Easy Exchange lets students and readers list books they no longer need, discover books others are offering, and request trades (book-for-book and/or book-for-pickup). The first release is a **campus-style book exchange**: users authenticate, publish listings, search and filter the catalog, and manage trade requests through a simple request lifecycle.

Out of scope for v1: payments, shipping labels, ISBN barcode scanning, in-app chat beyond request messages, and multi-category marketplace items (electronics, clothing, etc.).

---

## 2. Actors

| Actor | Description |
| --- | --- |
| Guest | Unauthenticated visitor. May browse public listings. |
| Registered user | Authenticated member who can list books, request trades, and respond to requests. |
| Listing owner | User who created a specific book listing. |
| Requester | User who opens a trade request against someone else’s listing. |
| Administrator (optional v1.1) | Staff who can moderate listings and disable abusive accounts. |

---

## 3. Functional requirements

### 3.1 User authentication

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-AUTH-01 | A guest can create an account with email, display name, and password. | Must |
| FR-AUTH-02 | The system must reject duplicate emails and passwords that fail the password policy (minimum 8 characters, at least one letter and one number). | Must |
| FR-AUTH-03 | A registered user can sign in with email and password and receive a session that persists across page reloads until sign-out or expiry. | Must |
| FR-AUTH-04 | A registered user can sign out, which invalidates the current session on the server. | Must |
| FR-AUTH-05 | Unauthenticated users who hit a protected action (create listing, send trade request) are redirected to sign-in with a return path. | Must |
| FR-AUTH-06 | A user can view and edit their profile (display name, campus/location optional text, bio). Email is not editable in v1. | Should |
| FR-AUTH-07 | Password reset via emailed one-time link. | Could (v1.1) |

### 3.2 Listing books

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-LIST-01 | An authenticated user can create a listing with: title, author, condition, description, cover image (optional), course/subject tag (optional), and desired trade notes (what they want in return). | Must |
| FR-LIST-02 | Allowed condition values: `New`, `Like New`, `Good`, `Acceptable`. | Must |
| FR-LIST-03 | Listing status values: `Available`, `Pending Trade`, `Exchanged`, `Withdrawn`. New listings start as `Available`. | Must |
| FR-LIST-04 | The owner can edit title, author, condition, description, image, tags, and trade notes while status is `Available`. | Must |
| FR-LIST-05 | The owner can withdraw a listing (`Withdrawn`). Withdrawn listings are hidden from public search. Open trade requests on that listing are cancelled with a notice to requesters. | Must |
| FR-LIST-06 | The owner cannot delete a listing that has a completed trade; it remains as a historical record with status `Exchanged`. | Must |
| FR-LIST-07 | Guests and users can open a listing detail page showing metadata, owner display name, status, and (if authenticated and not the owner) a “Request trade” action. | Must |
| FR-LIST-08 | A user can view a “My listings” page of books they own, filterable by status. | Must |
| FR-LIST-09 | Cover images are optional; if omitted, the UI shows a placeholder. Uploaded images must be JPEG/PNG/WebP, max 5 MB. | Should |

### 3.3 Search and filter

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-SRCH-01 | Guests and users can search listings by free-text query matching title, author, description, and tags. | Must |
| FR-SRCH-02 | Results can be filtered by condition, status (`Available` only for public catalog by default), and optional course/subject tag. | Must |
| FR-SRCH-03 | Results can be sorted by newest first (default), title A–Z, or condition. | Should |
| FR-SRCH-04 | Empty results show a helpful empty state, not a blank page. | Must |
| FR-SRCH-05 | Search is case-insensitive and trims whitespace. Queries longer than 100 characters are rejected with a validation message. | Must |
| FR-SRCH-06 | Pagination (or infinite scroll) so catalogs of 50+ listings remain usable. Page size default: 12. | Should |

### 3.4 Trade requests

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-TRD-01 | An authenticated user who is **not** the listing owner can create a trade request against an `Available` listing. | Must |
| FR-TRD-02 | A trade request includes: target listing, optional offered listing(s) owned by the requester (book-for-book), a message, and a proposed meetup/handoff note. | Must |
| FR-TRD-03 | A user cannot have two **open** requests (`Pending` or `Accepted`) on the same listing. | Must |
| FR-TRD-04 | Request statuses: `Pending`, `Accepted`, `Declined`, `Cancelled`, `Completed`. | Must |
| FR-TRD-05 | The listing owner is notified (in-app notification; email optional) when a request is created. | Must (in-app) |
| FR-TRD-06 | The owner can **accept** one pending request. Accepting sets that request to `Accepted`, the listing to `Pending Trade`, and auto-declines other pending requests on the same listing. | Must |
| FR-TRD-07 | The owner can **decline** a pending request. The listing stays `Available`. | Must |
| FR-TRD-08 | Either party can **cancel** a `Pending` or `Accepted` request. If the listing was `Pending Trade` and no other accepted request remains, status returns to `Available`. | Must |
| FR-TRD-09 | Either party can mark an `Accepted` request **completed**. Both parties must confirm completion, **or** (v1 simplification) the owner’s completion confirmation is sufficient. **v1 decision:** owner marks complete; requester is notified. Listing becomes `Exchanged`. | Must |
| FR-TRD-10 | Users have an inbox of sent and received requests, filterable by status. | Must |
| FR-TRD-11 | Request detail shows both parties, listing snapshot (title/author at request time), messages, and allowed actions for the current user and status. | Must |
| FR-TRD-12 | Users cannot request a trade on their own listing. | Must |

### 3.5 Notifications (minimum)

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-NTF-01 | In-app notifications for: new incoming request, request accepted/declined/cancelled, trade completed. | Must |
| FR-NTF-02 | Unread count shown in the header. Opening a notification marks it read. | Should |

---

## 4. Non-functional requirements

### 4.1 Usability

| ID | Requirement |
| --- | --- |
| NFR-USE-01 | Primary flows (sign in, create listing, search, send/respond to request) complete without referring to external docs. |
| NFR-USE-02 | Forms show inline validation before submit where practical (required fields, image type/size). |
| NFR-USE-03 | The UI is usable on a laptop viewport (1280×800) and a phone viewport (375×667). |
| NFR-USE-04 | Interactive controls have visible labels or accessible names (WCAG 2.2 AA as a target, not a formal audit in v1). |

### 4.2 Performance

| ID | Requirement |
| --- | --- |
| NFR-PERF-01 | Public catalog first contentful render target: under 2 seconds on a typical campus Wi-Fi connection in local/dev with ≤500 listings. |
| NFR-PERF-02 | Search and filter responses return within 500 ms for ≤1,000 listings in local development. |
| NFR-PERF-03 | Cover images are stored as files/object storage and served via URL; listing pages must not embed raw multi-megabyte blobs in JSON. |

### 4.3 Security

| ID | Requirement |
| --- | --- |
| NFR-SEC-01 | Passwords are stored with a modern one-way hash (e.g. Argon2 or bcrypt); never logged or returned in APIs. |
| NFR-SEC-02 | Sessions use HTTP-only, Secure (in production), SameSite cookies or equivalent; CSRF protection on cookie-authenticated mutating requests. |
| NFR-SEC-03 | Authorization is enforced on the server: only owners mutate their listings; only participants mutate a given trade request. |
| NFR-SEC-04 | Uploaded files are type-checked (MIME + extension); stored outside the web root or via a dedicated uploads route; filenames are generated, not user-supplied. |
| NFR-SEC-05 | User-generated text is escaped in the UI to prevent XSS. |
| NFR-SEC-06 | Secrets (database URL, auth secret) live in environment variables, not in the repo. |

### 4.4 Reliability and data

| ID | Requirement |
| --- | --- |
| NFR-REL-01 | Listing and request status transitions are transactional (no listing stuck `Pending Trade` with zero accepted requests). |
| NFR-REL-02 | The app degrades with a clear error page/message if the database is unreachable; it does not show stack traces to end users. |
| NFR-REL-03 | Core entities (users, listings, requests) persist across server restarts. |

### 4.5 Maintainability

| ID | Requirement |
| --- | --- |
| NFR-MAINT-01 | Specs in `specs/` remain the source of truth for v1 behavior until implementation notes say otherwise. |
| NFR-MAINT-02 | TypeScript throughout application code. Shared types for listing status and request status. |
| NFR-MAINT-03 | Environment-specific config via `.env.example` (no real secrets committed). |

### 4.6 Compliance / academic constraints

| ID | Requirement |
| --- | --- |
| NFR-CLS-01 | Planning artifacts (`specs/requirements.md`, `architecture.md`, `behavior.md`) exist before feature implementation. |
| NFR-CLS-02 | v1 does not handle payments or real money; exchanges are non-commercial trades. |

---

## 5. Data constraints (summary)

**User:** id, email (unique), password hash, display name, optional location, optional bio, timestamps.

**Listing:** id, owner id, title, author, condition, description, optional image URL, optional tags, trade notes, status, timestamps.

**Trade request:** id, listing id, requester id, optional offered listing id(s), message, handoff note, status, timestamps, listing title/author snapshot.

**Notification:** id, user id, type, related entity id, read flag, timestamp.

---

## 6. Acceptance criteria (v1 slice)

A release is acceptable for the assignment’s implementation phase when:

1. A new user can register, sign in, and sign out.
2. That user can create a book listing and see it on My Listings and in public search.
3. Another user can find it via search/filter and submit a trade request.
4. The owner can accept or decline; accept moves the listing to `Pending Trade`.
5. The owner can complete the trade; the listing becomes `Exchanged` and leaves the public catalog.
6. Unauthorized users cannot edit others’ listings or accept others’ incoming requests (verified by API or UI plus server checks).
