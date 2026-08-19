# EventHub — Booking Management Test Strategy

Generated: 2026-08-15
Scope: Booking Management (59 scenarios in `docs/test-scenarios.md`, TC-001–TC-510)
Input: `docs/test-scenarios.md` · Domain skill (`business-rules.md`, `api-reference.md`) · Backend
source (`bookingService.js`, `bookingRepository.js`, `bookingController.js`, `bookingValidator.js`,
`eventService.js`) · Frontend source (`app/bookings/**`, `components/bookings/BookingCard.jsx`,
`lib/hooks/useBookings.ts`, `lib/api/bookingsApi.js`, `lib/api/bookings.ts`, `lib/api/client.{js,ts}`)
· Existing tests: `tests/booking-management.spec.js` reappeared in the working tree since the
prior pass of this document (it was absent then) — 4 tests / 6 TC-IDs: TC-001, TC-002, TC-003,
TC-004, TC-006, TC-102. Findings from reading it are folded into §5.

> **Update**: `playwright-best-practices` skill is now available and has been read. It changes two
> things from the prior pass of this document, both corrected below: (1) it documents
> `page.route()` API mocking as an established project pattern for isolating UI states — this
> means most "Component" scenarios are achievable *today* with zero new tooling, not blocked on
> Vitest/RTL as previously stated (§1, §6); (2) it sanctions extended-timeout assertions for
> timed UI (the refund spinner) as *not* the `waitForTimeout` anti-pattern, which combined with
> Playwright's `page.clock` API (available since v1.45; this repo has `^1.58.2`) means the
> refund-eligibility scenarios can stay at E2E without paying a real 4-second wait per test —
> reversing the "push to Component" call made last pass (§4).

---

## 1. Testing Infrastructure — Current State (read this first)

The layer assignments below are the *ideal* ones per the decision rules. But right now the repo
can only execute **one** of the four layers:

| Layer | Tooling present? | Evidence |
|---|---|---|
| Unit | ❌ None | No Jest/Vitest/Mocha in `backend/package.json`. No test script beyond `node --check` (syntax only). |
| API/Integration | ❌ None | No Supertest or equivalent. `bookingService`/`bookingRepository` have zero test coverage today. |
| Component | ❌ None | No Vitest/Jest/React Testing Library in `frontend/package.json`. |
| E2E | ✅ Playwright | `@playwright/test` in root `package.json`; `playwright.config.ts` present; **`tests/` directory is currently empty** (0 spec files) |

This means the project is structurally an **ice-cream cone today** — not because anyone chose
E2E-heavy testing, but because E2E is the *only* layer that currently has a runner. Every
assignment to Unit/API/Component below is a **recommendation requiring new tooling**, not
something that can be run today without setup work. See §6 for what to add.

One more infra caveat that matters for the API-layer recommendations specifically:
`playwright.config.ts`'s `baseURL` is hardcoded to `https://eventhub.rahulshettyacademy.com`
(the hosted deployment) with no env override. Playwright's `request` fixture *can* hit API
endpoints directly without going through the UI (no new dependency needed — this is the
cheapest path to "API layer" coverage), but pointed at that same `baseURL` it would pollute
shared/production-like data. Any API-layer suite should run against a local backend
(`http://localhost:3001/api`) via a **separate Playwright project** or a dedicated
Jest+Supertest setup — not the existing UI project.

---

## 2. Target Distribution

| Layer | Count | % | Focus | Est. cost/test | Est. total runtime |
|---|---|---|---|---|---|
| Unit | 3 | 5% | Pure ref-generation logic | ~10ms | <1s |
| API/Integration | 29 | 49% | Business rules, validation, auth/ownership, seat math | ~200–500ms (real DB round trip) | ~10–15s |
| Component | 22 | 37% | Isolated UI states, rendering logic, conditional displays | ~50–150ms (RTL render, mocked network) | ~2–3s |
| E2E | 5 | 8% | Full-stack critical journeys only | ~3–8s (real browser + backend; TC-103 adds a real 4s wait) | ~25–40s |
| **Total** | **59** | 100% | | | ~40–60s |

**Why Unit is the smallest bucket here (not a red flag):** booking management is a CRUD/business-
rule-heavy domain, not an algorithmic one. Almost every rule (FIFO pruning, seat availability,
ownership checks) is inherently stateful — it reads/writes through `bookingRepository` /
`eventRepository` and can't be tested meaningfully without a database, so it belongs at the
API/Integration layer, not Unit. The three genuine Unit candidates are the only pure,
side-effect-free logic in the booking domain: booking-reference formatting. The pyramid principle
that *matters* here is still satisfied — E2E is deliberately the narrowest layer (5 of 59, 8%),
and nothing that can be checked without a browser is being checked with one.

---

## 3. Layer Assignments

### Happy Path

| TC | Title | Layer | Source |
|---|---|---|---|
| TC-001 | View bookings list | **Component** *(has redundant E2E today)* | `app/bookings/page.tsx` (`BookingsContent`), `components/bookings/BookingCard.jsx` — E2E exists: `tests/booking-management.spec.js:71` |
| TC-002 | View booking detail page | **Component** *(has redundant E2E today)* | `app/bookings/[id]/page.tsx` — E2E exists: `tests/booking-management.spec.js:89` |
| TC-003 | Cancel booking from detail page | **E2E** ✅ covered | `app/bookings/[id]/page.tsx` → `useCancelBooking` → `bookingsApi.js` → `DELETE /api/bookings/:id` — `tests/booking-management.spec.js:171` |
| TC-004 | Clear all bookings | **E2E** ✅ covered, but see caveat below | `app/bookings/page.tsx` (`handleClearAll`) — `tests/booking-management.spec.js:202`. Per source-level analysis this should be **failing** (see TC-312); could not be verified in this environment (no Chromium install — see §5). If this test is currently green in CI, that contradicts the TC-312 source analysis and needs reconciling before trusting either. |
| TC-005 | Back-to-bookings navigation | **Component** | `app/bookings/[id]/page.tsx` (link only) |
| TC-006 | Navigate via "View My Bookings" post-booking | **E2E** ✅ covered | Multi-page journey: booking flow → `/bookings` — `tests/booking-management.spec.js:121` |
| TC-007 | Lookup booking by ref (API) | **API** | `bookingController.getBookingByRef` → `bookingService.getBookingByRef` |

### Business Rules

| TC | Title | Layer | Source |
|---|---|---|---|
| TC-100 | FIFO — 10th booking prunes oldest, different event | **API** | `bookingService.createBooking`, `bookingRepository.findOldestUserBookingExcludingEvent` |
| TC-101 | FIFO — same-event fallback burns a seat | **API** | `bookingService.createBooking` (`sameEventFallback` → `eventRepository.decrementSeats`) |
| TC-102 | Booking ref prefix = event title first char | **Unit** | `bookingService.js` `randomRef()` *(not currently exported — see §5)*. **Currently implemented as E2E** in `tests/booking-management.spec.js:156` — a full login+book journey just to check a string prefix. Concrete instance of the "pure logic tested at E2E" anti-pattern; see §5. |
| TC-103 | Refund eligibility — qty=1 eligible (real 4s wait) | **E2E** | `RefundEligibility` component in `app/bookings/[id]/page.tsx` |
| TC-104 | Refund eligibility — qty>1 ineligible | **Component** | Same component, fake timers instead of real 4s wait |
| TC-105 | Refund spinner duration (~4s) | **Component** | Same component, fake timers |
| TC-106 | totalPrice = price × quantity | **API** | `bookingService.createBooking` (inline, not an extracted pure fn — see §5) |
| TC-107 | Bookings default page size = 10 | **API** | `bookingService.getBookings` |
| TC-108 | Cancel releases computed seats (dynamic events) | **API** | `bookingService.cancelBooking` + `eventService.withPersonalSeats` |
| TC-109 | "Clear all bookings" link always visible | **Component** | `app/bookings/page.tsx` |

### Security

| TC | Title | Layer | Source |
|---|---|---|---|
| TC-200 | Cross-user access → "Access Denied" (real 2-account journey) | **E2E** | Full session-isolation proof — cannot be faithfully simulated by mocking one account |
| TC-201 | Cross-user GET → 403 | **API** | `bookingService.getBookingById` (`findByIdOnly` + explicit check) |
| TC-202 | Cross-user DELETE → 404, not 403 | **API** | `bookingService.cancelBooking` (`findById(id, userId)` pre-scoped — dead `ForbiddenError` branch) |
| TC-207 | 403-vs-404 inconsistency (read vs cancel) | **API** | Same two functions, compared in one test |
| TC-203 | Unauth GET list → 401 | **API** | `authMiddleware.js` |
| TC-204 | Unauth GET detail → 401 | **API** | `authMiddleware.js` |
| TC-205 | Unauth DELETE-all → 401 | **API** | `authMiddleware.js` — also the exact failure mode behind TC-312 |
| TC-206 | Cross-user ref lookup → 403 | **API** | `bookingService.getBookingByRef` |
| TC-208 | Booking a private event you don't own → 404 | **API** | `eventRepository.findById(id, userId)` scoped lookup |

### Negative / Error

| TC | Title | Layer | Source |
|---|---|---|---|
| TC-300 | Nonexistent booking ID → "Booking not found" UI | **Component** | `app/bookings/[id]/page.tsx` (`isError`, `is403=false` branch) |
| TC-301 | GET nonexistent ID → 404 | **API** | `bookingService.getBookingById` |
| TC-302 | Insufficient seats → 400 | **API** | `bookingService.createBooking` (`InsufficientSeatsError`) |
| TC-303 | Book nonexistent event → 404 | **API** | `bookingService.createBooking` |
| TC-304 | Missing required fields → 400 | **API** | `bookingValidator.validateCreateBooking` |
| TC-305 | quantity ≤ 0 → 400 | **API** | `bookingValidator.js` |
| TC-306 | quantity > 10 → 400 | **API** | `bookingValidator.js` |
| TC-307 | Cancel already-cancelled booking → 404 | **API** | `bookingService.cancelBooking` |
| TC-308 | Bookings list — server-error empty state | **Component** | `app/bookings/page.tsx` (`isError` branch, mocked failed query) |
| TC-309 | Malformed email → 400 | **API** | `bookingValidator.js` (`.isEmail()`) |
| TC-310 | Phone with letters → 400 | **API** | `bookingValidator.js` (regex) |
| TC-311 | Non-integer eventId → 400 | **API** | `bookingValidator.js` (`.isInt()`) |
| TC-312 | Clear-all silently fails (missing auth header) | **API** | `lib/api/bookings.ts` + `lib/api/client.ts` (root-cause, precise header assertion) — UI symptom already covered by TC-004's E2E |

### Edge Cases

| TC | Title | Layer | Source |
|---|---|---|---|
| TC-400 | FIFO boundary — exactly 9, prefers different event | **API** | Same as TC-100, boundary variant |
| TC-401 | FIFO boundary — exactly 9, same event | **API** | Same as TC-101, boundary variant |
| TC-402 | quantity = 1 stepper boundary | **Component** | Event-detail quantity stepper (`+`/`-` disabled state) |
| TC-403 | quantity = 10 stepper boundary | **Component** | Same stepper, upper bound |
| TC-404 | Refund boundary — qty=2 ineligible | **Component** | `RefundEligibility`, fake timers |
| TC-405 | Booking ref collision retry | **Unit** | `bookingService.js` `generateUniqueRef()` *(needs export + `bookingRepository` mock — see §5)* |
| TC-406 | Clear-all with exactly 1 booking | **API** | `bookingService.clearAllBookings` — boundary variant of TC-312's root cause, not a new E2E |
| TC-407 | Pagination page 2 | **API** | `bookingService.getBookings` |
| TC-408 | Ref prefix — digit title | **Unit** | Same as TC-102 |

### UI State

| TC | Title | Layer | Source |
|---|---|---|---|
| TC-500 | Skeleton loading (bookings list) | **Component** | `BookingCardSkeleton` |
| TC-501 | Empty state — zero bookings | **Component** | `app/bookings/page.tsx` |
| TC-502 | Loading spinner (detail page) | **Component** | `app/bookings/[id]/page.tsx` |
| TC-503 | Cancel confirmation dialog appears | **Component** | `ConfirmDialog` usage in `BookingCard.jsx` / detail page |
| TC-504 | Dismiss dialog without confirming | **Component** | Same |
| TC-505 | Breadcrumb shows booking ref | **Component** | `app/bookings/[id]/page.tsx` |
| TC-506 | Cancel success — toast + redirect | **Component** | `handleCancel` onSuccess (mocked mutation) — real proof is TC-003's E2E |
| TC-507 | "Clearing…" button state | **Component** | `app/bookings/page.tsx` (`clearing` state) |
| TC-508 | Refund button hidden after result | **Component** | `RefundEligibility` state machine |
| TC-509 | "Access Denied" state (403) | **Component** | `app/bookings/[id]/page.tsx` (mocked 403 response) — real proof is TC-200's E2E |
| TC-510 | Pagination renders on multi-page | **Component** | `Pagination` usage in `app/bookings/page.tsx` |

---

## 4. Decision Rationale (contested / non-obvious assignments)

**Refund eligibility (TC-103/104/105/404/508) — split across E2E and Component, not all-E2E.**
The original scenario doc suggested E2E for all five. Each one waits a real 4 seconds
(`setTimeout(..., 4000)` in `RefundEligibility`) — running all five as E2E costs ~20s of pure
idle waiting per suite run for zero added confidence, since the logic (`quantity === 1 ?
'eligible' : 'ineligible'`) is identical regardless of which scenario exercises it. Keeping
**one** real E2E (TC-103, the eligible happy path) proves the full stack wires up correctly;
the other four move to Component tests with mocked/advanced timers, which test the same branches
in milliseconds. This is the textbook "pure logic pushed to E2E" anti-pattern from the skill's
checklist, corrected.

**Cross-user security (TC-200 vs TC-201/509) — same rule, three layers, different purpose each.**
TC-201 (API) is the primary enforcement proof: it's fast, deterministic, and directly asserts the
403 status/message with no UI in the way. TC-509 (Component) is a fast, isolated check that the
*rendering* branch (`is403` → "Access Denied" copy) is wired correctly, independent of a real
network round trip. TC-200 (E2E) is kept as genuine defense-in-depth *despite* the cost of a real
two-account login/logout cycle, because it's the only layer that proves actual session isolation
(two real JWTs, two real `localStorage` states, one real auth middleware) — something no amount
of mocking one account can substitute for. Rule 2 in `business-rules.md` ("cross-user access
returns 403 Forbidden") is arguably the single most safety-critical rule in this app, which
justifies paying for all three layers on it.

**Booking-reference generation (TC-102/405/408) — Unit, but blocked on a small refactor.**
`randomRef()` and `generateUniqueRef()` in `bookingService.js` are the *only* genuinely
low-level-testable logic in the booking domain (`randomRef` is pure; `generateUniqueRef` only
touches `bookingRepository.findByRef`, trivially mockable). But neither is currently exported —
only the `bookingService` object is. Until they're exported (or tested indirectly by mocking the
whole `bookingRepository` module and invoking `createBooking`), these three scenarios cannot
actually run as Unit tests. Flagged as a concrete, small prerequisite in §5/§6 rather than
silently downgrading them to API tests (which would work today, but permanently forfeits the
cheapest, fastest layer for the one place in this domain that supports it).

**FIFO pruning (TC-100/101/400/401) — API, not Unit, despite being "business logic."**
These involve creating up to 10 real booking rows, ordering by `createdAt`, and asserting which
row survives — behavior that lives across `bookingService` *and* `bookingRepository`'s Prisma
queries (`findOldestUserBookingExcludingEvent`, `countUserBookings`). Mocking the repository
to fake this would mean re-implementing Prisma's `orderBy`/`findFirst` semantics in the mock,
which tests the mock, not the code. Decision rule 5 ("could work at a lower layer? push it down")
doesn't apply here — there is no lower layer that doesn't defeat the purpose of the test.

**"Clear all bookings" (TC-004/312/406) — deliberately split symptom (E2E) from root cause (API).**
TC-004 stays E2E even though it's *expected to fail right now* — that's the point: an E2E test
here is a standing regression guard that will start failing the moment someone reintroduces this
bug (or, right now, immediately demonstrates it). TC-312 is a separate, precise API-layer test
that inspects the actual outgoing request (no `Authorization` header) and asserts the 401 —
useful for pinpointing the root cause without needing a browser at all once someone starts
fixing it. TC-406 (the "exactly 1 booking" edge case) doesn't need its own E2E — it's a boundary
variant of the same root cause already covered by TC-312, so it's assigned straight to API.

**UI loading/error states (TC-308/500/501/502) — Component with mocked responses, not E2E with
network throttling.** The original doc's "Component / E2E" dual-suggestion for these leans E2E
in practice (throttling a real network in Playwright is fiddly and flaky). Mocking the
`useBookings`/`useBooking` query result directly (React Query supports this cleanly) gets the
same `isLoading`/`isError` branches exercised deterministically and near-instantly.

**Quantity stepper boundaries (TC-402/403) — Component, not E2E.** The *business-rule* boundary
(quantity must be 1–10) is already covered API-side by TC-305/TC-306. What TC-402/403 actually
test is UI behavior — whether the `+`/`-` buttons disable at the boundary — which is a
render/interaction concern belonging in Component, not a reason to drive a full booking-creation
form through a real browser twice more.

---

## 5. Anti-Patterns Found

### In `tests/booking-management.spec.js` (existing file — now readable, was absent last pass)

1. **Pure logic tested at E2E — TC-102 (`tests/booking-management.spec.js:156-168`).** The test
   logs in, clears bookings, books a real event, and drives a full page navigation solely to
   assert `bookingRef` starts with `eventTitle[0].toUpperCase()`. That's the exact
   `randomRef()` string-formatting rule from `bookingService.js` — zero I/O, a few lines, and it
   is currently the single most expensive way this repo could possibly verify it (full browser +
   real backend + real DB write, to check a regex). Textbook instance of the skill's "pure logic
   tested at E2E" anti-pattern, not hypothetical. Fix: Unit-test `randomRef()` directly per §4/§6;
   keep the E2E only as incidental coverage (it already gets exercised for free inside TC-003/004
   which need a booking anyway).
2. **No E2E coverage for the app's most safety-critical rule.** The suite has zero tests
   touching cross-user access control (TC-200/201/207) — the rule `business-rules.md` calls out
   explicitly ("Cross-user access to bookings returns 403 Forbidden"). Per the skill's own
   checklist ("No E2E tests for critical flows — always need some"), this is a real gap, not a
   theoretical one: nothing in this repo today would catch a regression that let User B read or
   act on User A's booking.
3. **No coverage at all for the refund-eligibility feature** (TC-103/104/105/404/508) — a
   feature `business-rules.md` documents as its own numbered rule (§8) with a distinctive
   4-second spinner and two branches (eligible/ineligible). Not present anywhere in the existing
   suite.
4. **What *is* there is solid and worth preserving as the house style**: every test is
   self-contained (`login()` → `clearBookings()` → act → assert, matching `CLAUDE.md`'s
   guidance), no `page.waitForTimeout()` anywhere, selectors favor `data-testid`/`#id` per the
   documented convention, and the `login()`/`bookEvent()`/`clearBookings()` helpers are exactly
   the reusable pattern `CLAUDE.md` calls "established." New E2E specs (§6) should extend this
   file and reuse these helpers rather than reinventing them elsewhere.
5. **Unresolved discrepancy worth flagging explicitly**: TC-004 in this file asserts "Clear all
   bookings" succeeds end-to-end against the hosted deployment. Source-level analysis in the
   prior pass of this document (TC-312 in `docs/test-scenarios.md`) concluded this button is
   broken — `app/bookings/page.tsx` calls the `.ts` API client (`lib/api/bookings.ts` →
   `lib/api/client.ts`), which never attaches the JWT, so the underlying `DELETE /api/bookings`
   should 401. I attempted to run `TC-004` in this environment to settle it empirically
   (`node node_modules/@playwright/test/cli.js test ... --grep TC-004`) but Chromium isn't
   installed locally (`npx playwright install` was not run, and I did not install it unprompted
   given the download size) — so this is **not yet verified either way**. Whoever runs this suite
   next should check: if TC-004 is currently green, either the hosted deployment's frontend
   differs from what's in this working tree, or the TC-312 analysis has a flaw worth
   re-examining; if TC-004 is currently red, that's an unnoticed CI failure worth surfacing.

### Structural / doc-level

6. **No test infrastructure below E2E exists at all** (§1). This isn't a per-scenario anti-pattern,
   it's structural: 100% of currently-runnable tests must be E2E, which is the ice-cream-cone
   shape by construction, regardless of what this document recommends. Highest-priority fix.
7. **Original `docs/test-scenarios.md` over-assigns E2E to pure timer-driven Component logic** —
   all five refund-eligibility scenarios were marked E2E, four of which don't need a browser at
   all (see §4). At ~4s of real wait each, that's ~16s of avoidable suite time.
8. **`docs/test-scenarios.md` suggests "Unit" for TC-405 (ref collision retry) while the target
   functions aren't exported** — the suggestion is directionally right but not actually
   executable today. Needs the export change in §6 before it's a real Unit test, not just a label.
9. **`playwright.config.ts`'s `baseURL` is hardcoded to the hosted production-like deployment**
   with no local/env override (`CLAUDE.md` confirms this is intentional for the existing E2E
   suite). Any new API-layer suite must not casually reuse this config — it needs its own target
   (local backend), or API tests will create/delete real data against the shared hosted instance
   on every run.
10. **`totalPrice` (TC-106) and the FIFO oldest-selection logic are inline in `bookingService.js`
   rather than extracted into named, independently testable functions.** Not wrong, but it caps
   how far tests can be pushed down the pyramid — flagged as a nice-to-have refactor, not a
   blocker, since API-layer coverage of both is entirely adequate.

---

## 6. Recommendations (infra to add before `/generate-tests` can act on this)

1. **Backend: add Jest + Supertest** (`backend/package.json` devDependencies). Enables the 29 API
   scenarios to run as fast, isolated tests against a real (test) database, and the 3 Unit
   scenarios to run with `bookingRepository` mocked via `jest.mock(...)`.
2. **Export `randomRef` and `generateUniqueRef` from `bookingService.js`** (e.g., a named export
   alongside the default `bookingService` object, or move them to a small `bookingRefUtils.js`).
   Unblocks TC-102/405/408 as true Unit tests.
3. **Frontend: add Vitest + React Testing Library** (`frontend/package.json` devDependencies).
   Enables the 22 Component scenarios to render `BookingCard`, `BookingsContent`, and
   `BookingDetailPage` in isolation with mocked React Query data — no backend, no browser.
4. **Point new API tests at a local backend, not the hosted `baseURL`** — either a second
   Playwright project with its own `use.baseURL: 'http://localhost:3001/api'`, or (preferred,
   given #1) a standalone Jest+Supertest suite entirely outside the Playwright config.
5. **Extend the existing `tests/booking-management.spec.js`** (already covers TC-001/002/003/
   004/006/102) rather than creating a new file — add the two missing critical-flow E2E cases:
   TC-200 (cross-user "Access Denied," currently zero coverage of the app's most safety-critical
   rule) and TC-103 (refund eligibility happy path, currently zero coverage of a distinctly
   documented business rule). Reuse the existing `login()`/`bookEvent()`/`clearBookings()`
   helpers — they're solid. Also relocate TC-102's assertion out of its current E2E-only home
   into a Unit test once `randomRef()` is exported (§4/anti-pattern #1) — no need to delete the
   E2E line, just stop relying on it as the *only* check.
6. **Resolve the TC-004 discrepancy (anti-pattern #5) before trusting either the test or the bug
   report** — install Playwright's browsers (`npx playwright install chromium`) in an environment
   where that's acceptable, run `TC-004` against the hosted deployment, and reconcile the result
   with the TC-312 source-level analysis.
