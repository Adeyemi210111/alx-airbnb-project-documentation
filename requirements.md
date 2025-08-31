
# Backend Requirements Specification — Airbnb Clone
**Date:** 2025-08-31  
**Scope:** Technical & functional requirements for **all key backend features**: Authentication, Property Management, Search & Filtering, Booking, Payments, Reviews, Messaging, Notifications, Admin, and File Storage.  
**API Style:** RESTful JSON, JWT-based auth, RBAC (guest/host/admin).

---

## 0) Conventions
- **Auth:** `Authorization: Bearer <access_jwt>` on protected endpoints; refresh token for `/auth/refresh`.
- **Errors:** JSON `{ "error": "code", "message": "Readable text", "details": {} }` with proper HTTP status.
- **Pagination:** `page` (default 1), `page_size` (default 20, max 100). Responses include `items`, `page`, `page_size`, `total`.
- **Idempotency:** For state-creating ops (`POST /bookings`, payments), clients should send `Idempotency-Key: <uuid>`.
- **Roles:** guest = traveler, host = listing owner, admin = platform admin.

---

## 1) User Authentication

### Functional
- Register (guest/host), login, refresh, logout; optional OAuth (e.g., Google). Passwords are hashed (bcrypt/argon2).
- Access/refresh tokens; refresh rotation + revocation. Basic profile read/update.

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| POST | `/auth/register` | No | N/A | Create user (guest/host). |
| POST | `/auth/login` | No | N/A | Issue access/refresh tokens. |
| POST | `/auth/refresh` | Refresh | Any | Rotate refresh, new access. |
| POST | `/auth/logout` | Access | Any | Invalidate current session. |
| GET  | `/users/me` | Access | Any | Get own profile. |
| PATCH| `/users/me` | Access | Any | Update name, phone, photo_url, preferences. |

### Validation & Security
- `email` unique, valid; `password` >= 8 chars w/ letters & numbers; `role` in {guest,host,admin}; only admin can elevate to admin.
- Rate-limit `/auth/login` (e.g., 5/min/IP). CORS, origin checks, HTTPS-only cookies if used.

### Performance
- P95: login ≤ 400 ms; refresh ≤ 250 ms. Index on `User.email`.

---

## 2) Property Management

### Functional
- Hosts create/edit/delete listings; include name, description, location, `pricepernight`. Manage images, availability, amenities (optional).

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| POST | `/properties` | Access | host | Create listing. |
| GET  | `/properties` | Public/Access | any | List with filters & pagination. |
| GET  | `/properties/:id` | Public/Access | any | Detail page. |
| PATCH| `/properties/:id` | Access | owner/ admin | Update listing. |
| DELETE | `/properties/:id` | Access | owner/ admin | Soft delete/archive. |
| POST | `/properties/:id/images` | Access | owner/ admin | Upload image (returns URL). |
| DELETE | `/properties/:id/images/:imageId` | Access | owner/ admin | Remove image. |
| GET | `/properties/:id/availability` | Access | any | Public availability view. |
| PATCH | `/properties/:id/availability` | Access | owner/ admin | Block/unblock dates. |

### Validation
- `name` 3–200; `description` required; `pricepernight` > 0; only owner can modify; cannot delete if active future bookings (409).

### Performance
- P95: list ≤ 300 ms. Indexes on `host_id`, price, location/city; text search on name/description (if supported). Cache hot queries (TTL ≤ 5m).

---

## 3) Search & Filtering

### Functional
- Query properties by location, date availability, price range, guests, amenities; sorting & pagination.

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| GET | `/search/properties` | Public/Access | any | Wrapper or alias to `/properties` with rich filters. |

**Query Params (examples):** `city`, `country`, `start_date`, `end_date`, `min_price`, `max_price`, `guests`, `amenities=wifi,pool`, `sort=price_asc`.

### Validation & Performance
- Validate date range and numeric bounds. Use covering indexes on `(city,country)` and `(start_date,end_date)` join/exists against bookings. Avoid N+1 by preloading images/amenities summaries.

---

## 4) Booking System

### Functional
- Guests create bookings for available dates; prevent double booking. Lifecycle: `pending` → `confirmed` (after payment) → `completed` → `canceled`.
- Return payment init info (client secret/checkout URL). Hosts/guests can cancel per policy.

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| POST | `/bookings` | Access | guest | Create booking. |
| GET | `/bookings/:id` | Access | owner(guest/host)/admin | Booking detail. |
| GET | `/users/me/bookings` | Access | guest/host | List own bookings. |
| PATCH | `/bookings/:id/cancel` | Access | owner/admin | Cancel booking (policy checks). |

### Validation
- `end_date` > `start_date`; no overlap on same property against pending/confirmed; guest ≠ host; require idempotency key.

### Performance
- P95 create ≤ 500 ms (excl. external payment). Index on `(property_id, start_date, end_date)`; use transaction + unique/exclusion constraint to prevent overlaps.

---

## 5) Payments

### Functional
- Secure charges via Stripe/PayPal; store payment records; update booking status on webhook success; schedule **payouts** to hosts post-stay. Multi-currency support.

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| POST | `/payments/intent` | Access | guest | Init payment for booking (client secret / checkout URL). |
| POST | `/payments/webhook` | Provider | N/A | Receive provider events (signature verified). |
| GET  | `/payments/:id` | Access | owner(guest/host)/admin | Payment detail. |
| GET  | `/users/me/payments` | Access | guest/host | List payments for user. |
| GET  | `/payouts/:bookingId` | Access | host/admin | View payout status for a completed booking. |

### Validation & Security
- Verify booking belongs to user; amount = server-side calculation; verify webhook signatures; idempotency keys; never store PANs. Currency codes ISO-4217.

### Performance
- Webhook processing ≤ 300 ms P95; retries with idempotency. Background job for payouts with exponential backoff.

---

## 6) Reviews & Ratings

### Functional
- Booking-linked reviews (1–5 rating, comment). Host may respond. Prevent duplicate reviews per booking/author.

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| POST | `/reviews` | Access | guest | Create review for a completed booking. |
| GET  | `/properties/:id/reviews` | Public/Access | any | List property reviews. |
| POST | `/reviews/:id/respond` | Access | host | Host response to a review. |

### Validation
- Booking must exist, belong to reviewer, and be completed (`end_date` in past); rating 1..5; one review per (booking, author).

### Performance
- Paginate; precompute average rating per property (materialized view or nightly job) or compute with indexed aggregates.

---

## 7) Messaging

### Functional
- Authenticated two-way messaging between guest and host, tied to a property or booking.

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| POST | `/messages` | Access | guest/host | Send message `{recipient_id, body, property_id?|booking_id?}`. |
| GET  | `/conversations/:userId` | Access | owner/admin | Fetch conversation thread(s) with a user. |
| GET  | `/messages/:id` | Access | sender/recipient/admin | Single message. |

### Validation & Performance
- Sender ≠ recipient; body not empty; permission checks. Index `(sender_id, recipient_id, sent_at)`. Optional long-polling/WebSocket for realtime; else poll with ETags/If-Modified-Since.

---

## 8) Notifications

### Functional
- Email + in-app notifications for booking confirmations/cancellations, payment updates, and admin actions.

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| GET | `/notifications` | Access | any | List notifications (unread first). |
| PATCH | `/notifications/:id/read` | Access | owner | Mark as read. |

### Delivery
- Background worker consumes events (booking.created, payment.succeeded). Email via provider (SendGrid/Mailgun); in-app stored in DB.

### Performance
- Queue latency P95 ≤ 2s; retries with DLQ. Idempotent handlers.

---

## 9) Admin

### Functional
- Monitor and manage users, listings, bookings, payments; disputes; metrics.

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| GET | `/admin/users` | Access | admin | List/search users. |
| PATCH | `/admin/users/:id` | Access | admin | Suspend/role changes (no self-demotion). |
| GET | `/admin/properties` | Access | admin | List/search properties. |
| PATCH | `/admin/properties/:id` | Access | admin | Moderate property (hide/restore). |
| GET | `/admin/bookings` | Access | admin | List/search bookings. |
| GET | `/admin/payments` | Access | admin | List/search payments. |
| GET | `/admin/metrics` | Access | admin | Platform metrics. |

### Security
- Admin actions audited (`who`, `when`, `what`); read-only replicas for heavy analytics queries.

---

## 10) File Storage (Images)

### Functional
- Store property images and profile photos using local storage for implementation (S3/Cloudinary compatible later).

### Endpoints
| Method | Path | Auth | Roles | Description |
|---|---|---|---|---|
| POST | `/upload` | Access | host/guest | Upload image; returns `{url}`. |
| DELETE | `/upload` | Access | owner/admin | Delete by URL or key. |

### Validation & Performance
- Validate MIME type, size (e.g., ≤ 5MB), dimensions; virus scan optional; generate thumbnails; serve via CDN or static server. Asynchronous processing for large images.

---

## 11) Cross-Cutting (Security, Observability, Testing, Perf)

- **Security:** bcrypt/argon2; HTTPS; HSTS; rate limiting; input validation; RBAC; secrets in vault; no raw card data.
- **Observability:** structured logs with correlation IDs; metrics (req rate, latency p50/p95/p99, errors); tracing (optional).
- **Testing:** unit, integration, API contract; booking overlap tests; webhook idempotency tests.
- **Performance Targets (P95):** Public reads ≤ 300 ms; writes ≤ 500 ms (excl. external calls). Use DB indexes noted above and Redis caching for hot paths.

---

## 12) Example Error Codes
- `400` validation_error, `401` unauthorized, `403` forbidden, `404` not_found, `409` conflict, `422` rule_violation, `429` rate_limited, `500` internal.
