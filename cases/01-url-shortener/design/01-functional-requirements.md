# Functional Requirements — URL Shortener

> Status: **Accepted**. Scope decisions settled 2026-11-12.
> Source: *System Design Interview*, Vol. 1, "Design a URL Shortener".

## 1. Purpose

Turn a long URL into a short code and redirect anyone who opens the short URL to the original address. Links are created once and opened many times, so the system is read-heavy.

## 2. Actors

| Actor | Description |
|---|---|
| **Client** | Anyone calling the API to create a short URL. Anonymous: there are no accounts. |
| **Visitor** | Anyone (browser, bot, HTTP client) that opens a short URL. |

## 3. Use cases

### UC-1 Shorten a URL

1. The client submits a long URL and, optionally, an expiration time.
2. The system validates the URL (absolute `http`/`https`, at most 2,048 characters).
3. The system generates a new, unique short code and stores the mapping.
4. The system returns the short URL, the short code and the expiration time, if one was set.

**Alternative flows**
- Invalid URL or invalid expiration (in the past, malformed) → rejected with a validation error. Nothing is stored.

### UC-2 Redirect a short URL

1. The visitor opens `/{code}`.
2. The system looks up the mapping.
3. The system answers with an HTTP redirect to the long URL with `302 Found` (see [D-1](#d-1-click-analytics-and-301-vs-302)).

**Alternative flows**
- Unknown code → `404 Not Found`.
- Code exists but has expired → `410 Gone`. It never redirects.

## 4. Functional requirements

| ID | Requirement |
|---|---|
| FR-1 | The system shall accept a long URL and return a short URL that maps to it. Each request creates a new code, even for a long URL that was shortened before. |
| FR-2 | Every short code shall be unique. A code is never reassigned to a different long URL, even after it expires. |
| FR-3 | Short codes shall use only URL-safe characters (`[0-9a-zA-Z]`). Their length is set in the architecture, based on the capacity estimates in `02-non-functional-requirements.md`. |
| FR-4 | Opening a short URL shall redirect to the stored long URL with `302 Found`. |
| FR-5 | The client may set an optional expiration time when creating a link. A link with no expiration never expires. |
| FR-6 | An expired link shall never redirect. It returns `410 Gone`. |
| FR-7 | Invalid input shall be rejected with a descriptive validation error. |
| FR-8 | Unknown codes shall return `404 Not Found`. |

## 5. Scope decisions

| Question | Decision | Rationale |
|---|---|---|
| Custom aliases | **Out** | It adds a second write path (user-chosen keys, collision handling, reserved words) that teaches nothing new about the core mechanisms. Under *Simple first* it can come back later through an ADR. |
| Expiration / TTL | **In, optional** | It's cheap to add (one optional field) and it exercises interesting mechanics later: expiry checks on the read path, cache invalidation and background cleanup. How expired rows get purged is decided in the architecture, not here. |
| Click analytics | **Out**, redirects use 302 (see D-1) | It needs its own pipeline (event capture, aggregation) that is better saved for a later case built around stream processing. |
| Same long URL → same code? | **No**, every request gets a new code (see D-2) | Keeps the write path a simple insert and avoids clashing with TTL. |
| Delete / update links | **Out** | Mappings stay immutable, which keeps caching trivial (no invalidation on writes) and fits the read-heavy profile. Expiration is the only way a link stops working. |
| Users / authentication | **Out** | This is a lab non-goal. All clients are anonymous. Abuse controls such as rate limiting are quality concerns and belong in the NFRs if they're needed. |

## 6. Key decisions

### D-1 Click analytics and 301 vs 302

- **Decision:** analytics **out**, redirect with **302 Found**.
- **Why 302 even without analytics:** browsers cache a 301 permanently. Once that happens, (a) an **expired link keeps redirecting** from the visitor's cache, which breaks FR-6, and (b) the service and its telemetry stop seeing real redirect traffic, which goes against *Measure everything*. With 302, every redirect reaches the system, so the load hits the cache and the database, which is exactly what the experiments are meant to measure.
- **Cost:** more read traffic than with 301. That's acceptable, because the read path is the thing under study.
- **Record as:** an ADR (`adr/0001-redirect-status-code.md`), linked to the latency and throughput scenarios in the NFRs.

### D-2 Same long URL → same short code?

- **Decision:** **no deduplication**. Every shorten request creates a new code.
- **Why:**
  - Deduplication requires a lookup by long URL on every write, which means a second index (or a hash of the long URL) that must stay consistent with the primary key, plus a race to handle when two concurrent requests submit the same URL.
  - It conflicts with TTL: what should happen when the same URL is submitted with a different expiration?
  - Without users there's no privacy or ownership argument either way.
  - It keeps the write path a simple insert, in line with *Simple first*.
- **Trade-off accepted:** storage grows with duplicates, and retrying a shorten request after a timeout creates a second code (shorten is not idempotent). If idempotency matters later, an `Idempotency-Key` header solves it without dedup on the URL.
- **Record as:** an ADR (`adr/0002-no-long-url-deduplication.md`), since it shapes the data model and the code generation strategy.

## 7. Out of scope (summary)

Custom aliases, click analytics, link deletion or update, users and authentication, link previews, URL safety scanning, and a web UI.
