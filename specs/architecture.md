# Easy Exchange — Architecture

## Architecture specification

**Product:** Easy Exchange (book exchange)  
**Status:** Planning (stack and structure only; no application code in this phase)

This document records the intended v1 technical design so implementation can follow the specs without inventing structure on the fly.

---

## 1. Design goals

- One deployable web app a small student team can run locally.
- Clear split between UI, HTTP/API, domain rules (status machines), and persistence.
- Server-enforced authorization (never “hide the button” as the only check).
- Match `specs/requirements.md` entities and status values exactly.

---

## 2. Tech stack selection

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript | Shared types for statuses, DTOs, and less runtime surprise. |
| Framework | Next.js (App Router) | UI + API routes in one project; SSR for catalog SEO/first load; fits a class web app. |
| UI | React 19 + Tailwind CSS | Fast layout work; utility CSS without a large component vendor lock-in. |
| Auth | Auth.js (NextAuth) Credentials provider | Email/password as specified; session cookies; well-documented for Next.js. |
| Password hashing | bcrypt (or Argon2 if the runtime makes it easy) | Meets NFR-SEC-01. |
| ORM / DB access | Prisma | Schema-first models aligned with requirements; migrations. |
| Database | SQLite in development; PostgreSQL-ready schema | Zero-ops local setup; same Prisma models later. |
| File uploads | Local `uploads/` in dev (ignored by git); optional S3-compatible later | FR-LIST-09 without cloud accounts on day one. |
| Validation | Zod | Parse API bodies and form data; one schema per use case. |
| Testing (implementation phase) | Vitest for domain/status helpers; Playwright for listing + trade happy paths | Behavior file becomes the test outline. |

**Explicitly not chosen for v1:** separate Express microservice, GraphQL, Redis, Elasticsearch, Docker Swarm, mobile native apps.

---

## 3. High-level architecture

```text
┌─────────────────────────────────────────────┐
│                 Browser                      │
│  App Router pages + client components        │
└──────────────────┬──────────────────────────┘
                   │ HTTPS / same-origin
┌──────────────────▼──────────────────────────┐
│              Next.js server                  │
│  • RSC pages / Server Actions or Route Handlers
│  • Auth.js session                           │
│  • Zod validation                            │
│  • Domain services (listings, trades)        │
│  • Prisma                                    │
└───────┬──────────────────────────┬──────────┘
        │                          │
        ▼                          ▼
   SQLite / Postgres          Disk / object store
   (users, listings,          (cover images)
    requests, notifications)
```

**Request path (mutating):** UI form → Server Action or `POST /api/...` → session check → Zod parse → domain service (status rules) → Prisma transaction → optional notification rows → redirect or JSON.

**Read path:** Server component loads Prisma data (or cached catalog query) → render. Search query string drives filters.

---

## 4. Folder structure (intended repo)

```text
easy-exchange/
├── specs/                      # This planning set (source of truth)
│   ├── requirements.md
│   ├── architecture.md
│   └── behavior.md
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── public/                     # Static assets (placeholders, favicon)
├── uploads/                    # Dev-only images (gitignored)
├── src/
│   ├── app/                    # Next.js App Router
│   │   ├── layout.tsx
│   │   ├── page.tsx            # Public catalog / home
│   │   ├── login/
│   │   ├── register/
│   │   ├── books/
│   │   │   ├── page.tsx        # Search results
│   │   │   ├── new/page.tsx    # Create listing
│   │   │   └── [id]/page.tsx   # Listing detail
│   │   ├── me/
│   │   │   ├── listings/page.tsx
│   │   │   └── requests/page.tsx
│   │   ├── requests/
│   │   │   └── [id]/page.tsx
│   │   └── api/                # Route handlers if not using Actions only
│   │       ├── auth/[...nextauth]/
│   │       └── uploads/
│   ├── components/             # Reusable UI
│   │   ├── layout/             # Header, nav, footer
│   │   ├── books/              # BookCard, SearchFilters, ListingForm
│   │   ├── trades/             # RequestForm, RequestStatusBadge, RequestActions
│   │   └── ui/                 # Button, Input, Alert (thin wrappers)
│   ├── lib/
│   │   ├── db.ts               # Prisma client singleton
│   │   ├── auth.ts             # Auth.js config
│   │   ├── session.ts          # getCurrentUser helpers
│   │   └── utils.ts
│   ├── domain/                 # Pure rules (easy to unit test)
│   │   ├── listingStatus.ts
│   │   └── tradeRequestStatus.ts
│   ├── services/               # Orchestration + Prisma
│   │   ├── listingService.ts
│   │   ├── searchService.ts
│   │   ├── tradeService.ts
│   │   └── notificationService.ts
│   └── types/                  # Shared TS types / enums
├── .env.example
├── package.json
└── README.md                   # Run instructions (implementation phase)
```

Implementation must not start until this planning phase is accepted; the tree above is the **target**, not files to create now except `specs/`.

---

## 5. Component breakdown

### 5.1 UI components

| Component | Responsibility |
| --- | --- |
| `SiteHeader` | Logo, catalog link, search entry, auth state, notification unread count, My Listings / Requests. |
| `SearchFilters` | Query input, condition, tag, sort; writes URL search params. |
| `BookCard` | Cover, title, author, condition, status chip; links to detail. |
| `BookGrid` | Responsive list of `BookCard`; empty state. |
| `ListingForm` | Create/edit fields; client + server validation. |
| `ListingDetail` | Full metadata, owner identity, trade CTA or owner tools. |
| `RequestForm` | Offered book picker, message, handoff note. |
| `RequestActions` | Accept / decline / cancel / complete, gated by role + status. |
| `RequestList` | Sent vs received tabs; status filter. |
| `NotificationBell` | Dropdown of recent notifications. |
| `AuthForm` | Login and register variants. |

Pages compose these components; pages stay thin.

### 5.2 Domain modules

**`listingStatus.ts`**

- Allowed transitions: `Available` → `Pending Trade` | `Withdrawn`  
  `Pending Trade` → `Available` | `Exchanged` | `Withdrawn` (withdraw cancels open requests)  
  `Exchanged` terminal  
  `Withdrawn` terminal for public catalog
- Helpers: `isPubliclySearchable(status)`, `canEdit(status)`, `canRequestTrade(status)`.

**`tradeRequestStatus.ts`**

- `Pending` → `Accepted` | `Declined` | `Cancelled`  
  `Accepted` → `Completed` | `Cancelled`  
  `Declined`, `Cancelled`, `Completed` terminal
- Helpers: `actionsFor(role, status)` returning allowed action names.

These modules must not import Prisma (pure functions) so behavior.md can be unit-tested later.

### 5.3 Services

| Service | Responsibility |
| --- | --- |
| `listingService` | Create, update, withdraw, get by id, get by owner; enforce owner checks. |
| `searchService` | Case-insensitive query, filters, sort, pagination; default `Available` only. |
| `tradeService` | Create request, accept (transaction: accept one, decline others, listing → `Pending Trade`), decline, cancel, complete (listing → `Exchanged`). |
| `notificationService` | Insert notification rows; mark read; unread count. |
| `uploadService` | Validate MIME/size; store; return public URL. |

### 5.4 Data model (Prisma-oriented)

```text
User          1 ─── * Listing
User          1 ─── * TradeRequest (as requester)
Listing       1 ─── * TradeRequest
User          1 ─── * Notification
Listing       * ─── * Listing (optional offered books via join table TradeOffer)
```

Indexes: `Listing.status`, `Listing.createdAt`, full-text or `contains` on title/author for v1; unique `User.email`.

---

## 6. Auth and authorization

- **Authentication:** Auth.js Credentials; JWT or database sessions (database sessions preferred if Prisma adapter is used).
- **Authorization matrix:**

| Action | Guest | Other user | Owner | Requester |
| --- | --- | --- | --- | --- |
| Browse/search `Available` | Yes | Yes | Yes | Yes |
| Create listing | No | Yes | — | — |
| Edit/withdraw listing | No | No | Yes | No |
| Create trade request | No | Yes (not owner) | No | — |
| Accept/decline | No | No | Yes | No |
| Cancel | No | No | Yes | Yes |
| Complete (v1) | No | No | Yes | No |

Enforcement lives in services, not only in components.

---

## 7. Key API / server operations

Whether implemented as Server Actions or REST, the operations are:

| Operation | Input | Result |
| --- | --- | --- |
| `register` | email, name, password | User + session |
| `signIn` / `signOut` | credentials | Session |
| `createListing` | listing fields + optional file | Listing |
| `updateListing` | id + fields | Listing |
| `withdrawListing` | id | Listing + cancelled requests |
| `searchListings` | q, filters, page | Page of listings |
| `createTradeRequest` | listingId, offerIds, message, handoff | Request + notification |
| `acceptTradeRequest` | requestId | Request + listing + declined siblings |
| `declineTradeRequest` | requestId | Request |
| `cancelTradeRequest` | requestId | Request + maybe listing restore |
| `completeTradeRequest` | requestId | Request + listing `Exchanged` |

Errors: `401` unauthenticated, `403` forbidden, `404` missing, `409` conflict (duplicate open request, illegal status transition), `422` validation.

---

## 8. Environment and config

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL` | Prisma connection |
| `AUTH_SECRET` | Session signing |
| `UPLOAD_DIR` | Local upload path (dev) |

Documented in `.env.example` during implementation. Never commit real secrets.

---

## 9. Implementation order (after specs sign-off)

1. Scaffold Next.js + Prisma + User model + auth pages.  
2. Listings CRUD + My Listings + detail.  
3. Search/filter on catalog.  
4. Trade request lifecycle + notifications.  
5. Image upload.  
6. Tests mapped to `behavior.md`.

Do not skip domain status helpers; they are the core of FR-TRD-*.
