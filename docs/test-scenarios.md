# EventHub — Booking Management Test Scenarios

Generated: 2026-03-06 · Refreshed: 2026-08-15
Scope: Booking Management (Flow 4 — View, Cancel, Clear, Refund Eligibility)

**Refresh note**: Re-verified against live source: `backend/src/services/bookingService.js`,
`bookingRepository.js`, `bookingController.js`, `bookingRoutes.js`, `bookingValidator.js`,
`errorHandler.js`, `authMiddleware.js`, and on the frontend `app/bookings/page.tsx`,
`app/bookings/[id]/page.tsx`, `components/bookings/BookingCard.jsx`, `lib/hooks/useBookings.ts`,
`lib/api/bookingsApi.js`, `lib/api/bookings.ts`, `lib/api/client.js`, `lib/api/client.ts`.
Two real discrepancies were found and are captured below: **TC-202/TC-207** (cross-user
cancel returns 404, not 403) and **TC-312** (Clear All Bookings is functionally broken).

---

## Happy Path

### TC-001: View bookings list with existing bookings
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User is logged in; user has at least one confirmed booking
**Steps**:
1. Navigate to `/bookings`
2. Observe the list of booking cards rendered
**Expected Results**: Each booking card (`#booking-card`) displays booking reference, status badge, booking ID (`#booking-id`), event title, event date, quantity, city, "booked on" date, total price, and a "View Details" link; a "Cancel Booking" button (`#cancel-booking-btn`) appears only when `status === 'confirmed'`
**Business Rule**: Flow 4 — Manage Bookings; `BookingCard.jsx`
**Suggested Layer**: E2E

---

### TC-002: View single booking detail page
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User is logged in; user has at least one confirmed booking
**Steps**:
1. Navigate to `/bookings`
2. Click "View Details" on any booking card
3. Observe the booking detail page at `/bookings/:id`
**Expected Results**: Page shows event details (title, category, date, venue, city), customer details (name, email, phone), payment summary (tickets, price per ticket, total paid), a Refund section, booking metadata (booked-on date, `#{id}`), booking reference + status badge in the header, and breadcrumb "My Bookings / {bookingRef}"
**Business Rule**: Booking model fields; Flow 4; `app/bookings/[id]/page.tsx`
**Suggested Layer**: E2E

---

### TC-003: Cancel a single booking from the detail page
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User is logged in; user has at least one confirmed booking
**Steps**:
1. Navigate to `/bookings/:id`
2. Click "Cancel Booking" button
3. Confirm in the dialog by clicking "Yes, cancel it"
4. Observe redirect and bookings list
**Expected Results**: `useCancelBooking` fires `DELETE /api/bookings/:id` via `bookingsApi.js` (axios client — Authorization header attached correctly); success toast "Booking cancelled successfully" appears; user is redirected to `/bookings`; cancelled booking no longer appears in the list; `bookings` and `events` React Query caches are invalidated
**Business Rule**: Booking cancellation deletes the record; seats released for dynamic events (computed)
**Suggested Layer**: E2E

---

### TC-004: Clear all bookings from the bookings list page
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User is logged in; user has at least one booking
**Steps**:
1. Navigate to `/bookings`
2. Click "Clear all bookings" link
3. Confirm the browser `confirm()` dialog
4. Observe the page after clearing
**Expected Results (intended)**: All bookings are removed; page shows empty state "No bookings yet" with "Browse Events" button; `DELETE /api/bookings` returns `{ deleted: N }`
**Actual current behavior**: **This is broken — see TC-312.** The button calls `bookingsApi.clearAll()` from the `.ts` client (`lib/api/bookings.ts` → `lib/api/client.ts`), which never attaches the `Authorization` header, so the request 401s server-side and nothing is cleared. `handleClearAll` has no `catch`, so this fails silently — button briefly reads "Clearing…", then the list is unchanged.
**Business Rule**: `DELETE /api/bookings` clears all user bookings; `clearAllBookings` service method
**Suggested Layer**: E2E

---

### TC-005: Navigate back to bookings list from detail page
**Category**: Happy Path
**Priority**: P2
**Preconditions**: User is on a booking detail page
**Steps**:
1. Click "← Back to My Bookings" button at bottom of detail page
**Expected Results**: User is navigated to `/bookings`
**Business Rule**: UI navigation flow
**Suggested Layer**: E2E

---

### TC-006: Navigate to bookings via "View My Bookings" after completing a booking
**Category**: Happy Path
**Priority**: P1
**Preconditions**: User just completed a booking (confirmation card shown)
**Steps**:
1. After booking confirmation, click "View My Bookings" link
2. Observe the bookings page
**Expected Results**: User lands on `/bookings` and the newly created booking appears in the list
**Business Rule**: Flow 3 → Flow 4 navigation
**Suggested Layer**: E2E

---

### TC-007: Lookup booking by reference via API
**Category**: Happy Path
**Priority**: P1
**Preconditions**: User is authenticated; user has a booking with known `bookingRef`
**Steps**:
1. Send `GET /api/bookings/ref/:ref` with valid JWT and own booking ref
**Expected Results**: 200 response with full booking data including nested event
**Business Rule**: `GET /api/bookings/ref/:ref` endpoint
**Suggested Layer**: API

---

## Business Rules

### TC-100: FIFO pruning — 10th booking replaces oldest booking from a different event
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User has exactly 9 bookings (all for different events); user has JWT token
**Steps**:
1. Note the oldest booking ID
2. Create a new booking (10th) for a different event via `POST /api/bookings`
3. Retrieve all user bookings
**Expected Results**: Total booking count remains 9; the oldest booking is deleted; the new booking is present
**Business Rule**: Max 9 bookings per user; FIFO pruning prefers deleting from a different event (`findOldestUserBookingExcludingEvent`)
**Suggested Layer**: API

---

### TC-101: FIFO pruning — same-event fallback permanently burns a seat
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User has exactly 9 bookings all for the SAME event; enough seats remain
**Steps**:
1. Create a 10th booking for the same event
2. Retrieve the event's available seats
**Expected Results**: Oldest booking is deleted; new booking is created; `availableSeats` decremented by the new booking's quantity (seat permanently burned via `eventRepository.decrementSeats`)
**Business Rule**: `sameEventFallback` path in `bookingService.createBooking`
**Suggested Layer**: API

---

### TC-102: Booking reference first character matches event title first character
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User is logged in; event with known title exists (e.g., "Tech Conference Bangalore")
**Steps**:
1. Book the event
2. Read the `bookingRef` from the confirmation card or API response
**Expected Results**: `bookingRef` starts with the uppercase first character of the event title (e.g., "T-XXXXXX" for "Tech Conference")
**Business Rule**: `randomRef` function: `prefix = (eventTitle?.[0] ?? 'E').toUpperCase()`; Rule 7
**Suggested Layer**: E2E / API

---

### TC-103: Refund eligibility — single ticket booking is eligible
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User has a booking with quantity = 1
**Steps**:
1. Navigate to `/bookings/:id` for the single-ticket booking
2. Click "Check eligibility for refund?" (`#check-refund-btn`)
3. Wait for spinner to disappear (4 seconds)
4. Read the refund result
**Expected Results**: `#refund-result` shows green "Eligible for refund. Single-ticket bookings qualify for a full refund."
**Business Rule**: Rule 8 — `quantity === 1` → eligible; `RefundEligibility` component in `app/bookings/[id]/page.tsx`
**Suggested Layer**: E2E

---

### TC-104: Refund eligibility — multi-ticket booking is NOT eligible
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User has a booking with quantity > 1 (e.g., 3 tickets)
**Steps**:
1. Navigate to `/bookings/:id` for the multi-ticket booking
2. Click "Check eligibility for refund?"
3. Wait for spinner to disappear (4 seconds)
4. Read the refund result
**Expected Results**: `#refund-result` shows red "Not eligible for refund. Group bookings (3 tickets) are non-refundable." with correct quantity displayed
**Business Rule**: Rule 8 — quantity > 1 → not eligible
**Suggested Layer**: E2E

---

### TC-105: Refund eligibility spinner shows for approximately 4 seconds
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User is on a booking detail page
**Steps**:
1. Click "Check eligibility for refund?"
2. Immediately check for spinner
3. Observe when spinner disappears and result appears
**Expected Results**: `#refund-spinner` is visible immediately after clicking (status `'checking'`); spinner disappears and `#refund-result` appears after exactly 4000ms (`setTimeout(..., 4000)`)
**Business Rule**: Rule 8 — `RefundEligibility` status state machine: `idle → checking → eligible/ineligible`
**Suggested Layer**: E2E / Component
---

### TC-106: Total price is calculated as price × quantity
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User books an event with known price
**Steps**:
1. Book an event (e.g., price $1499, quantity 3)
2. View the booking detail page
**Expected Results**: "Total Paid" shows $4,497 (1499 × 3); `totalPrice` in API response equals `parseFloat(event.price) * quantity`
**Business Rule**: Rule 9 — `totalPrice = event.price × quantity` (`bookingService.createBooking`)
**Suggested Layer**: E2E / API

---

### TC-107: Bookings list/API default page size is 10
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User has bookings
**Steps**:
1. Send `GET /api/bookings` with no `limit` param
2. Load `/bookings` in the UI
**Expected Results**: API defaults `limit = 10` (`bookingService.getBookings`); frontend explicitly requests `limit: 10` in `useBookings({ page, limit: 10 })`; response includes `pagination.limit`, `pagination.totalPages`
**Business Rule**: Rule 4 — max 9 bookings per user (so pagination rarely triggers in practice, but the mechanism itself is testable)
**Suggested Layer**: API

---

### TC-108: Cancelling a booking releases seat count for dynamic events (computed, not decremented server-side)
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User has a dynamic (user-created) event with a booking
**Steps**:
1. Note the current available seats for the event (computed: `totalSeats - sum(user's booking quantities)`)
2. Cancel the booking for that event
3. Re-fetch the event detail
**Expected Results**: Available seats increase by the cancelled booking's quantity. Note this is purely a side effect of the booking row being deleted (`bookingRepository.delete`) — `cancelBooking` does **not** call `eventRepository.incrementSeats`; the swagger comment on `DELETE /bookings/:id` ("atomically restores the released seats") is misleading/inaccurate documentation for dynamic events, though the net user-visible effect is the same
**Business Rule**: Rule 6 — dynamic events compute seats as `totalSeats - sum(user's booking quantities)`
**Suggested Layer**: API / E2E

---

### TC-109: Bookings list shows "Clear all bookings" link whenever bookings exist
**Category**: Business Rule
**Priority**: P2
**Preconditions**: User has at least one booking
**Steps**:
1. Navigate to `/bookings`
2. Look for "Clear all bookings" link
**Expected Results**: Link is visible in the top-right of the page header (unconditional — not gated on booking count); a small "Do this often for clean test data." hint is shown below it. See TC-312 for the fact that clicking it does not currently work.
**Business Rule**: Flow 4 — UI always shows clear option; `app/bookings/page.tsx`
**Suggested Layer**: E2E / Component

---

## Security

### TC-200: Cross-user booking access returns "Access Denied" (UI)
**Category**: Security
**Priority**: P0
**Preconditions**: Two test accounts exist (rahulshetty1@gmail.com and rahulshetty1@yahoo.com); User A has a booking
**Steps**:
1. Log in as User A, create a booking, note the booking ID
2. Log out (clear localStorage JWT)
3. Log in as User B
4. Navigate to `/bookings/:userA_booking_id`
**Expected Results**: Page shows "Access Denied" title and "You are not authorized to view this booking." description (`error.status === 403` branch in `BookingDetailPage`)
**Business Rule**: Rule 2 — cross-user access returns 403; frontend renders "Access Denied" on 403 response
**Suggested Layer**: E2E

---

### TC-201: Cross-user booking access returns 403 via API
**Category**: Security
**Priority**: P0
**Preconditions**: User A has a booking; User B has a valid JWT
**Steps**:
1. Send `GET /api/bookings/:userA_booking_id` with User B's JWT
**Expected Results**: HTTP 403; response body contains "You are not authorized to view this booking"
**Business Rule**: `bookingService.getBookingById` — unscoped `findByIdOnly` lookup, then explicit `booking.userId !== userId` → `ForbiddenError`
**Suggested Layer**: API

---

### TC-202: Cross-user booking cancellation returns 404 (not 403) via API
**Category**: Security
**Priority**: P0
**Preconditions**: User A has a booking; User B has a valid JWT
**Steps**:
1. Send `DELETE /api/bookings/:userA_booking_id` with User B's JWT
**Expected Results**: HTTP 404 ("Booking with id X not found"); booking is NOT deleted from the database. (This is the correct current behavior — not 403; see TC-207 for why.)
**Business Rule**: `bookingService.cancelBooking` calls `bookingRepository.findById(id, userId)`, a Prisma `findFirst` already scoped to `{ id, userId }` — a cross-user booking simply doesn't match and returns `null`, so `NotFoundError` fires before the `booking.userId !== userId` → `ForbiddenError` check on the next line is ever reached (dead code)
**Suggested Layer**: API

---

### TC-207: Cross-user access is inconsistent between read (403) and cancel (404) endpoints
**Category**: Security
**Priority**: P2
**Preconditions**: User A has a booking; User B has a valid JWT
**Steps**:
1. Send `GET /api/bookings/:userA_booking_id` with User B's JWT → note status code
2. Send `DELETE /api/bookings/:userA_booking_id` with User B's JWT → note status code
**Expected Results**: GET returns 403; DELETE returns 404. The underlying security guarantee holds either way (User B can neither view nor delete User A's booking), but the status code/message is inconsistent across the two endpoints — worth a product decision on whether cancel should also 403 (would require switching `cancelBooking` to an unscoped lookup + explicit check, exercising the currently-dead `ForbiddenError` branch)
**Business Rule**: Compare `bookingService.getBookingById` vs `bookingService.cancelBooking`
**Suggested Layer**: API

---

### TC-203: Unauthenticated access to bookings list returns 401
**Category**: Security
**Priority**: P0
**Preconditions**: No valid JWT
**Steps**:
1. Send `GET /api/bookings` without Authorization header
**Expected Results**: HTTP 401; `{ success: false, error: 'Unauthorized' }`
**Business Rule**: `authMiddleware` — `router.use(authMiddleware)` in `bookingRoutes.js` applies to all `/api/bookings` routes
**Suggested Layer**: API

---

### TC-204: Unauthenticated access to booking detail returns 401
**Category**: Security
**Priority**: P0
**Preconditions**: No valid JWT
**Steps**:
1. Send `GET /api/bookings/:id` without Authorization header
**Expected Results**: HTTP 401; "Unauthorized"
**Business Rule**: `authMiddleware`
**Suggested Layer**: API

---

### TC-205: Unauthenticated DELETE /api/bookings returns 401
**Category**: Security
**Priority**: P0
**Preconditions**: No valid JWT
**Steps**:
1. Send `DELETE /api/bookings` without Authorization header
**Expected Results**: HTTP 401
**Business Rule**: `authMiddleware`; `clearAllBookings` requires authenticated user. Related to TC-312 — the UI's own "Clear all bookings" button also sends this same request with no Authorization header (bug, not intentional testing of this path), so it hits exactly this 401 response
**Suggested Layer**: API

---

### TC-206: Cross-user booking lookup by ref returns 403
**Category**: Security
**Priority**: P1
**Preconditions**: User A has a booking with known ref; User B has a valid JWT
**Steps**:
1. Send `GET /api/bookings/ref/:userA_ref` with User B's JWT
**Expected Results**: HTTP 403; "You do not own this booking"
**Business Rule**: `bookingService.getBookingByRef` — ownership check (unscoped `findByRef` + explicit check, same pattern as `getBookingById`)
**Suggested Layer**: API

---

### TC-208: Booking a private event owned by another user returns 404, not the event's data
**Category**: Security
**Priority**: P1
**Preconditions**: User A has a private dynamic (user-created) event; User B has a valid JWT and knows/guesses the event's ID
**Steps**:
1. Send `POST /api/bookings` as User B with `eventId` set to User A's private event ID
**Expected Results**: HTTP 404 "Event with id X not found" — same message/status as a genuinely non-existent event ID, so the endpoint doesn't leak whether the event exists but is just inaccessible
**Business Rule**: `eventRepository.findById(id, userId)` scopes to `{ id, OR: [{ isStatic: true }, { userId }] }`; `bookingService.createBooking` throws `NotFoundError` when this returns null, indistinguishable from a truly missing event
**Suggested Layer**: API

### TC-209: Clearing bookings removes only the authenticated user's records
**Category**: Security
**Priority**: P0
**Preconditions**: User A and User B each have at least one booking; both accounts have valid JWTs
**Steps**:
1. Record one booking ID for each user
2. Send `DELETE /api/bookings` with User A's JWT
3. Retrieve bookings as User A and User B
**Expected Results**: User A receives `{ deleted: N }` and has no bookings; User B's booking remains present and unchanged
**Business Rule**: `clearAllBookings` calls `deleteAllForUser(userId)`; booking data is isolated per user
**Suggested Layer**: API

---

## Negative / Error

### TC-300: Navigate to non-existent booking ID shows "Booking not found"
**Category**: Negative
**Priority**: P1
**Preconditions**: User is logged in
**Steps**:
1. Navigate to `/bookings/99999` (ID that does not exist)
**Expected Results**: Page shows "Booking not found" and "This booking doesn't exist or may have been cancelled." with "View My Bookings" button
**Business Rule**: `bookingService.getBookingById` throws `NotFoundError` → API returns 404; frontend's `isError` branch with `is403 === false` renders this state
**Suggested Layer**: E2E

---

### TC-301: GET /api/bookings/:id with non-existent ID returns 404
**Category**: Negative
**Priority**: P1
**Preconditions**: User is authenticated
**Steps**:
1. Send `GET /api/bookings/99999` with valid JWT
**Expected Results**: HTTP 404; error message "Booking with id 99999 not found"
**Business Rule**: `bookingService.getBookingById` — `NotFoundError`
**Suggested Layer**: API

---

### TC-302: Create booking with insufficient seats returns 400
**Category**: Negative
**Priority**: P0
**Preconditions**: User is authenticated; event has 0 personal available seats (all booked by this user)
**Steps**:
1. Send `POST /api/bookings` with `quantity: 1` for a fully-booked event
**Expected Results**: HTTP 400; "Only 0 seat(s) available, but 1 requested"
**Business Rule**: `bookingService.createBooking` — `InsufficientSeatsError` when `personalAvailable < quantity`
**Suggested Layer**: API

---

### TC-303: Create booking for non-existent event returns 404
**Category**: Negative
**Priority**: P1
**Preconditions**: User is authenticated
**Steps**:
1. Send `POST /api/bookings` with `eventId: 99999`
**Expected Results**: HTTP 404; "Event with id 99999 not found"
**Business Rule**: `bookingService.createBooking` — event lookup fails → `NotFoundError`
**Suggested Layer**: API

---

### TC-304: Create booking with missing required fields returns 400
**Category**: Negative
**Priority**: P1
**Preconditions**: User is authenticated
**Steps**:
1. Send `POST /api/bookings` with missing `customerName`, `customerEmail`, or `customerPhone`
**Expected Results**: HTTP 400; `{ error: 'Validation failed', details: [{ field, message }, ...] }`
**Business Rule**: `bookingValidator.validateCreateBooking`
**Suggested Layer**: API

---

### TC-305: Create booking with quantity = 0 or negative returns 400
**Category**: Negative
**Priority**: P1
**Preconditions**: User is authenticated
**Steps**:
1. Send `POST /api/bookings` with `quantity: 0`
2. Send `POST /api/bookings` with `quantity: -1`
**Expected Results**: HTTP 400; validation error for both cases ("Quantity must be an integer between 1 and 10")
**Business Rule**: `body('quantity').isInt({ min: 1, max: 10 })`
**Suggested Layer**: API

---

### TC-306: Create booking with quantity > 10 returns 400
**Category**: Negative
**Priority**: P1
**Preconditions**: User is authenticated
**Steps**:
1. Send `POST /api/bookings` with `quantity: 11`
**Expected Results**: HTTP 400; validation error
**Business Rule**: quantity must be 1–10
**Suggested Layer**: API

---

### TC-307: Cancel a booking that has already been cancelled returns 404
**Category**: Negative
**Priority**: P1
**Preconditions**: User is authenticated; a booking exists
**Steps**:
1. Delete the booking via `DELETE /api/bookings/:id`
2. Attempt to delete the same booking again
**Expected Results**: HTTP 404; "Booking with id X not found"
**Business Rule**: `cancelBooking` uses `bookingRepository.findById` — not found after deletion
**Suggested Layer**: API

---

### TC-308: Bookings page shows error state when server is unreachable
**Category**: Negative
**Priority**: P2
**Preconditions**: Backend server is down or returns 500
**Steps**:
1. Navigate to `/bookings` with backend unavailable
**Expected Results**: `isError` branch renders: "Couldn't load bookings", "Failed to connect to the server. Please try again.", and a "Retry" button that calls `refetch()`
**Business Rule**: `isError` branch in `BookingsContent` component
**Suggested Layer**: Component / E2E

---

### TC-309: Create booking with malformed (not missing) customer email returns 400
**Category**: Negative
**Priority**: P2
**Preconditions**: User is authenticated
**Steps**:
1. Send `POST /api/bookings` with `customerEmail: "not-an-email"`
**Expected Results**: HTTP 400; validation error `{ field: "customerEmail", message: "Customer email must be a valid email address" }`
**Business Rule**: `bookingValidator.validateCreateBooking` — `body('customerEmail').isEmail()`
**Suggested Layer**: API

---

### TC-310: Create booking with customer phone containing letters returns 400
**Category**: Negative
**Priority**: P2
**Preconditions**: User is authenticated
**Steps**:
1. Send `POST /api/bookings` with `customerPhone: "call-me-maybe"`
**Expected Results**: HTTP 400; validation error `{ field: "customerPhone", message: "Customer phone must contain only digits and +, -, spaces, or parentheses" }`
**Business Rule**: `bookingValidator.validateCreateBooking` — `body('customerPhone').matches(/^[0-9+\-\s()]+$/)`
**Suggested Layer**: API

---

### TC-311: Create booking with non-integer eventId returns 400
**Category**: Negative
**Priority**: P2
**Preconditions**: User is authenticated
**Steps**:
1. Send `POST /api/bookings` with `eventId: "abc"`
**Expected Results**: HTTP 400; validation error `{ field: "eventId", message: "Event ID must be a positive integer" }`
**Business Rule**: `bookingValidator.validateCreateBooking` — `body('eventId').isInt({ min: 1 })`
**Suggested Layer**: API

### TC-313: Create booking with a one-character customer name returns 400
**Category**: Negative
**Priority**: P2
**Preconditions**: User is authenticated; a bookable event exists
**Steps**:
1. Send `POST /api/bookings` with `customerName: "A"` and otherwise valid fields
**Expected Results**: HTTP 400; validation details identify `customerName` and state that it must be at least 2 characters
**Business Rule**: `bookingValidator.validateCreateBooking` — customer name minimum length is 2
**Suggested Layer**: API

### TC-314: Create booking with a phone number shorter than 10 characters returns 400
**Category**: Negative
**Priority**: P2
**Preconditions**: User is authenticated; a bookable event exists
**Steps**:
1. Send `POST /api/bookings` with `customerPhone: "123456789"` and otherwise valid fields
**Expected Results**: HTTP 400; validation details identify `customerPhone` and state that it must be at least 10 digits
**Business Rule**: `bookingValidator.validateCreateBooking` — phone minimum length is 10 characters
**Suggested Layer**: API

---

### TC-312: "Clear all bookings" button silently fails — missing Authorization header (discovered defect)
**Category**: Negative
**Priority**: P0
**Preconditions**: User is logged in with a valid JWT in `localStorage`; user has at least one booking
**Steps**:
1. Navigate to `/bookings`
2. Open browser devtools → Network tab
3. Click "Clear all bookings" and confirm the browser dialog
4. Inspect the outgoing `DELETE /api/bookings` request and its response
**Expected Results**: Request has **no** `Authorization` header (unlike every other booking request in the app); server responds 401 `{ success: false, error: 'Unauthorized' }`; no bookings are removed; button briefly shows "Clearing…" then reverts; **no error toast or any user-visible feedback appears** — the failure is completely silent from the UI's perspective
**Business Rule**: `app/bookings/page.tsx`'s `handleClearAll` calls `bookingsApi.clearAll()` imported from `@/lib/api/bookings` (the `.ts` client) because `lib/api/bookingsApi.js` (the `.js` client used everywhere else in the booking flow) has no `clearAll` method. `lib/api/bookings.ts` → `lib/api/client.ts` is a bare `fetch` wrapper with no request interceptor, so it never attaches `localStorage`'s `eventhub_token`. This contradicts CLAUDE.md's note that the `.ts` API set is unused dead code — for this one action it is live and broken. `handleClearAll` also has no `catch`, so the thrown error is swallowed with only a `finally` resetting the button state.
**Suggested Layer**: E2E / API (verify network request headers directly)

---

## Edge Cases

### TC-400: Exactly 9 bookings — adding a 10th prunes oldest from a DIFFERENT event (preferred)
**Category**: Edge Case
**Priority**: P0
**Preconditions**: User has exactly 9 bookings across multiple events
**Steps**:
1. Note the ID of the oldest booking (different event from the new booking's event)
2. Create a new (10th) booking for Event X
3. Check the bookings list
**Expected Results**: Count stays at 9; oldest booking (different event) is gone; new booking is present
**Business Rule**: `findOldestUserBookingExcludingEvent` preferential pruning in `bookingService.createBooking`
**Suggested Layer**: API

---

### TC-401: Exactly 9 bookings all from same event — 10th triggers same-event fallback and burns seat
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User has 9 bookings all for Event X
**Steps**:
1. Create a new booking for Event X (10th)
2. Re-fetch Event X's available seats
**Expected Results**: Oldest booking removed; new booking created; `availableSeats` is permanently decremented by the new quantity (seat burned via `eventRepository.decrementSeats`)
**Business Rule**: `sameEventFallback = true` → `decrementSeats` called in `bookingService.createBooking`
**Suggested Layer**: API

---

### TC-402: Booking with quantity = 1 (minimum) — full happy path
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User is logged in; event has available seats
**Steps**:
1. Navigate to event detail page
2. Leave quantity at 1 (default minimum)
3. Fill customer form and confirm booking
**Expected Results**: Booking created with `quantity: 1`; `totalPrice = price × 1`; booking ref generated
**Business Rule**: quantity boundary: 1 is minimum
**Suggested Layer**: E2E

---

### TC-403: Booking with quantity = 10 (maximum)
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User is logged in; event has >= 10 available seats
**Steps**:
1. Navigate to event detail; click "+" 9 times to reach quantity 10
2. Fill form and confirm booking
**Expected Results**: Booking created with `quantity: 10`; `totalPrice = price × 10`; increment button disabled at 10
**Business Rule**: quantity boundary: 10 is maximum; UI should prevent going above 10
**Suggested Layer**: E2E

---

### TC-404: Refund eligibility boundary — quantity = 2 is NOT eligible (just above threshold)
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User has a booking with quantity = 2
**Steps**:
1. Navigate to booking detail
2. Click "Check eligibility for refund?"
3. Wait 4 seconds
**Expected Results**: Result shows "Not eligible for refund. Group bookings (2 tickets) are non-refundable."
**Business Rule**: Rule 8 — threshold is `quantity === 1`; quantity = 2 is the first ineligible value
**Suggested Layer**: E2E

---

### TC-405: Booking reference uniqueness — collision retry mechanism
**Category**: Edge Case
**Priority**: P2
**Preconditions**: Many bookings exist with the same event title prefix (stress scenario)
**Steps**:
1. Create many bookings for events starting with the same letter
2. Verify each `bookingRef` is unique in DB
**Expected Results**: All booking references are unique; no duplicates; after 10 failed retries, `generateUniqueRef` falls back to `${prefix}-${Date.now().toString(36).toUpperCase().slice(-8)}`
**Business Rule**: `generateUniqueRef` — up to 10 retries, then timestamp fallback
**Suggested Layer**: Unit

---

### TC-406: Clear all bookings when only one booking exists
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User has exactly 1 booking
**Steps**:
1. Navigate to `/bookings`
2. Click "Clear all bookings" and confirm
**Expected Results (intended)**: Booking is deleted; page shows empty state; `DELETE /api/bookings` returns `{ deleted: 1 }`
**Actual current behavior**: Same as TC-312 — the request 401s before reaching the server (no Authorization header), so nothing is deleted regardless of how many bookings exist
**Business Rule**: `clearAllBookings` — `deleteAllForUser` returns count of deleted records
**Suggested Layer**: E2E / API

---

### TC-407: Pagination on bookings list (API) — page 2 with partial results
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User has more than the default page limit of bookings visible in API
**Steps**:
1. Send `GET /api/bookings?page=2&limit=5`
**Expected Results**: Returns page 2 results; `pagination.page = 2`; `data` array contains at most 5 items
**Business Rule**: Pagination behavior in `bookingService.getBookings`
**Suggested Layer**: API

---

### TC-408: Event title starting with a number — booking ref prefix is uppercase of that character
**Category**: Edge Case
**Priority**: P2
**Preconditions**: An event exists whose title starts with a digit (e.g., "100 Days Festival")
**Steps**:
1. Book the event
2. Check the `bookingRef`
**Expected Results**: `bookingRef` starts with "1-XXXXXX" (digit is used as-is, `toUpperCase()` has no effect on digits)
**Business Rule**: `randomRef` — `prefix = (eventTitle?.[0] ?? 'E').toUpperCase()`
**Suggested Layer**: API / Unit

---

## UI State

### TC-500: Bookings list shows skeleton loading state while fetching
**Category**: UI State
**Priority**: P1
**Preconditions**: User navigates to `/bookings` (slow network or first load)
**Steps**:
1. Navigate to `/bookings` with throttled network
2. Observe the page before data loads
**Expected Results**: 5 `BookingCardSkeleton` placeholders are shown while `isLoading = true`; no real booking data yet
**Business Rule**: `isLoading` branch in `BookingsContent` — `Array.from({ length: 5 })`
**Suggested Layer**: Component / E2E

---

### TC-501: Bookings list shows empty state when user has no bookings
**Category**: UI State
**Priority**: P1
**Preconditions**: User is logged in with zero bookings
**Steps**:
1. Navigate to `/bookings`
**Expected Results**: Empty state renders with "No bookings yet", "You haven't booked any events yet. Browse upcoming events and grab your tickets!" description, and "Browse Events" button linking to `/events`
**Business Rule**: `bookings.length === 0` branch in `BookingsContent`
**Suggested Layer**: E2E / Component

---

### TC-502: Booking detail page shows loading spinner while fetching
**Category**: UI State
**Priority**: P2
**Preconditions**: User navigates to `/bookings/:id` on slow network
**Steps**:
1. Navigate to `/bookings/:id` with throttled network
2. Observe the page before data loads
**Expected Results**: Full-screen centered spinner (`Spinner size="lg"`) is visible while `isLoading = true`
**Business Rule**: `isLoading` branch in `BookingDetailPage`
**Suggested Layer**: Component

---

### TC-503: Cancel booking confirmation dialog appears before deletion
**Category**: UI State
**Priority**: P0
**Preconditions**: User is on a booking detail page (or bookings list)
**Steps**:
1. Click "Cancel Booking" button
2. Observe dialog
**Expected Results**: `ConfirmDialog` appears with title "Cancel this booking?", a description mentioning the booking ref and seat count (e.g., "Cancelling {ref} will release {N} seat(s) back to the event. This cannot be undone."), and a "Yes, cancel it" confirm button
**Business Rule**: Two-step confirmation prevents accidental cancellations
**Suggested Layer**: E2E / Component

---

### TC-504: Cancel booking dialog close without confirming does NOT cancel
**Category**: UI State
**Priority**: P1
**Preconditions**: User is on a booking detail page
**Steps**:
1. Click "Cancel Booking"
2. Click the close/dismiss button on the dialog (not "Yes, cancel it")
3. Observe booking status
**Expected Results**: Dialog closes (`onClose` sets `confirm = false`); booking remains in the list; no API call made
**Business Rule**: `handleCancel` only runs on confirm
**Suggested Layer**: E2E

---

### TC-505: Booking detail breadcrumb displays the booking reference
**Category**: UI State
**Priority**: P2
**Preconditions**: User navigates to a valid booking detail page
**Steps**:
1. Navigate to `/bookings/:id`
2. Observe the breadcrumb nav at the top
**Expected Results**: Breadcrumb shows "My Bookings / {bookingRef}" where `bookingRef` is rendered in a monospace span
**Business Rule**: Breadcrumb uses `booking.bookingRef`
**Suggested Layer**: E2E

---

### TC-506: Cancel booking success — toast and redirect
**Category**: UI State
**Priority**: P0
**Preconditions**: User confirms booking cancellation from the detail page
**Steps**:
1. Confirm cancellation in the dialog
2. Observe page transition and notifications
**Expected Results**: Success toast "Booking cancelled successfully" appears; user is redirected to `/bookings`
**Business Rule**: `onSuccess` callback in `handleCancel` (`app/bookings/[id]/page.tsx`)
**Suggested Layer**: E2E

---

### TC-507: "Clear all bookings" button shows "Clearing…" while in progress
**Category**: UI State
**Priority**: P2
**Preconditions**: User has bookings
**Steps**:
1. Click "Clear all bookings" and confirm dialog
2. Observe the button state while request is in flight
**Expected Results**: Button text changes to "Clearing…" and is disabled (`disabled:opacity-50`) during the API call. Note: per TC-312, the underlying request 401s almost immediately, so this loading state is very brief and is followed by silent failure rather than success — the button visuals themselves work correctly even though the action doesn't
**Business Rule**: `clearing` state variable in `BookingsContent`
**Suggested Layer**: Component / E2E

---

### TC-508: Refund eligibility — "Check eligibility" button hidden after result shown
**Category**: UI State
**Priority**: P2
**Preconditions**: User is on a booking detail page in idle refund state
**Steps**:
1. Click "Check eligibility for refund?"
2. Wait for result to appear
**Expected Results**: After status transitions `idle → checking → eligible/ineligible`, the initial button is no longer visible; spinner replaces it during check; result card replaces spinner after 4 seconds
**Business Rule**: `RefundEligibility` component status state machine
**Suggested Layer**: E2E / Component

---

### TC-509: Booking detail shows "Access Denied" state for 403 errors
**Category**: UI State
**Priority**: P0
**Preconditions**: Another user's booking ID is known
**Steps**:
1. Log in as User B
2. Navigate to `/bookings/:userA_booking_id`
3. Observe the rendered state
**Expected Results**: `EmptyState` with title "Access Denied" and description "You are not authorized to view this booking." renders (not "Booking not found")
**Business Rule**: Frontend checks `error.status === 403` to differentiate Access Denied vs Not Found
**Suggested Layer**: E2E

---

### TC-510: Bookings page pagination UI renders when total exceeds page size
**Category**: UI State
**Priority**: P2
**Preconditions**: API returns `pagination.totalPages > 1`
**Steps**:
1. Navigate to `/bookings` with enough bookings to trigger multi-page response
2. Observe pagination controls
**Expected Results**: `Pagination` component renders with `currentPage`/`totalPages` from the API response; clicking next page updates URL `?page=N` via `router.push` and loads next page of bookings
**Business Rule**: Pagination in `BookingsContent` driven by `pagination` from API response
**Suggested Layer**: E2E / Component

---

# EventHub — Create Events Test Scenarios

Scope: Create Events (Flow 5 — Create an authenticated user's private event from the Admin UI or API)

**Source discrepancy note**: The domain describes `category` and `city` as fixed value sets, but
`validateCreateEvent` only checks that both are non-empty strings. The scenarios below record the
implemented behavior where it differs from the domain expectation. The live UI uses
`eventsApi.js` and Axios `client.js`, which attaches `localStorage.eventhub_token`.

## Happy Path

### TC-008: Create an event with all supported fields from the Admin UI
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User is logged in; `/admin/events` is accessible; a future datetime, non-negative price, and positive seat count are available
**Steps**:
1. Navigate to `/admin/events`
2. Fill `#event-title-input`, `#admin-event-form textarea`, `getByLabel('Category')`, `getByLabel('City')`, `getByLabel('Venue')`, `getByLabel('Event Date & Time')`, `getByLabel('Price ($)')`, `getByLabel('Total Seats')`, and optional `getByLabel('Image URL (optional)')`
3. Click `#add-event-btn`
4. Observe the toast and `[data-testid="event-table-row"]`
**Expected Results**: `POST /api/events` returns 201 with `success: true`, `message: "Event created successfully"`; the UI shows `Event created!`, resets the form, invalidates the events query, and the new private event appears with the supplied values, `availableSeats === totalSeats`, `isStatic === false`, and the current user's `userId`
**Business Rule**: Flow 5; `eventService.createEvent` initializes available seats from total seats
**Suggested Layer**: E2E

### TC-009: Create an event through the API
**Category**: Happy Path
**Priority**: P0
**Preconditions**: User has a valid JWT
**Steps**:
1. Send `POST /api/events` with valid title, description, category, venue, city, future ISO 8601 `eventDate`, `price`, `totalSeats`, and optional `imageUrl`
2. Read the response body
3. Send `GET /api/events/:id` using the same JWT
**Expected Results**: POST returns 201 and the created event; GET returns the same event, including `availableSeats` equal to `totalSeats`, `isStatic: false`, and ownership scoped to the creator
**Business Rule**: `POST /api/events`; user-created events are private to their creator
**Suggested Layer**: API

### TC-010: Create an event with no description or image URL
**Category**: Happy Path
**Priority**: P1
**Preconditions**: User is authenticated; all required fields are valid
**Steps**:
1. Submit the Admin Event form with description and image URL blank
2. Inspect the created event in the table or via `GET /api/events/:id`
**Expected Results**: Creation succeeds; `description` is stored as an empty string and `imageUrl` is `null`; no validation error appears
**Business Rule**: Description and image URL are optional
**Suggested Layer**: E2E / API

### TC-011: Create events in each UI category
**Category**: Happy Path
**Priority**: P1
**Preconditions**: User is logged in; the form is empty
**Steps**:
1. Create one valid event selecting each `getByLabel('Category')` option: Conference, Concert, Sports, Workshop, and Festival
2. Inspect each created row
**Expected Results**: Each request succeeds and preserves the selected category in the response and table; the category select exposes exactly the five options listed by `EventForm`
**Business Rule**: Supported category choices in `EventForm.CATEGORIES`
**Suggested Layer**: E2E / Component

### TC-012: Newly created event is visible only to its creator
**Category**: Happy Path
**Priority**: P1
**Preconditions**: User A and User B both have valid accounts; User A can create events
**Steps**:
1. Create an event as User A and record its ID
2. Fetch `GET /api/events` as User A and User B
3. Fetch `GET /api/events/:id` as both users
**Expected Results**: User A sees the event in list and detail; User B does not see it in the list and receives 404 for the detail request
**Business Rule**: Static events are shared; dynamic events are private to their creator
**Suggested Layer**: API

## Business Rules

### TC-110: Sixth dynamic event is retained without pruning
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User has exactly five dynamic events and any number of static events
**Steps**:
1. Create a sixth valid dynamic event
2. Fetch `GET /api/events?limit=100`
**Expected Results**: All six user-created events are present; no dynamic event is deleted; static events do not count toward this limit
**Business Rule**: Maximum six user-created events per account
**Suggested Layer**: API

### TC-111: Seventh dynamic event prunes the oldest dynamic event
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User has six dynamic events created in a known order
**Steps**:
1. Record the oldest event ID and title
2. Create a seventh valid dynamic event
3. Fetch `GET /api/events?limit=100`
**Expected Results**: Six dynamic events remain; the oldest dynamic event is gone and the new event is present
**Business Rule**: FIFO pruning via `countUserDynamic` and `findOldestUserDynamic`; `MAX_USER_DYNAMIC_EVENTS = 6`
**Suggested Layer**: API

### TC-112: Event creation initializes availability from total seats
**Category**: Business Rule
**Priority**: P0
**Preconditions**: User is authenticated; valid payload has `totalSeats: 25`
**Steps**:
1. Create the event through `POST /api/events`
2. Read its response and then `GET /api/events/:id`
**Expected Results**: Persisted `availableSeats` starts at 25 and does not inherit another event's availability; personal booking subtraction happens only when events are read
**Business Rule**: `eventService.createEvent` sets available seats from parsed total seats
**Suggested Layer**: API

### TC-113: Numeric form values are sent and stored with correct types
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User is logged in; valid form values are available
**Steps**:
1. Enter decimal price `12.50` and integer seats `25`
2. Submit `#admin-event-form`
3. Inspect the POST payload and response
**Expected Results**: UI sends a number from `parseFloat` and an integer from `parseInt`; API persists price `12.50` and seats `25`, not strings
**Business Rule**: `EventForm.handleSubmit` and `eventService.createEvent` normalize numeric fields
**Suggested Layer**: E2E / API

### TC-114: Created event is included in the creator's paginated results
**Category**: Business Rule
**Priority**: P1
**Preconditions**: User is authenticated; a valid event can be created
**Steps**:
1. Create an event
2. Request `GET /api/events?page=1&limit=10`
3. Open `/events` and inspect `getByTestId('event-card')`
**Expected Results**: The event is eligible for the creator's list, pagination metadata is consistent, and its personal `availableSeats` is returned; static events remain visible
**Business Rule**: Repository ownership clause is `isStatic: true OR userId`; list defaults to page 1 and API limit 10
**Suggested Layer**: API / E2E

## Security

### TC-210: Unauthenticated Create Event API request returns 401
**Category**: Security
**Priority**: P0
**Preconditions**: No Authorization header or token is invalid
**Steps**:
1. Send `POST /api/events` with an otherwise valid payload and no Bearer token
2. Repeat with `Authorization: Bearer invalid-token`
**Expected Results**: Both requests return HTTP 401 with the auth middleware's Unauthorized response; no event is created
**Business Rule**: `router.use(authMiddleware)` protects every event route
**Suggested Layer**: API

### TC-211: Admin Create Event page requires authentication
**Category**: Security
**Priority**: P0
**Preconditions**: Browser has no `eventhub_token`
**Steps**:
1. Navigate directly to `/admin/events`
2. Observe the resulting route and network calls
**Expected Results**: `AuthGuard` redirects to `/login`; the form is not usable and no unauthenticated POST is attempted
**Business Rule**: Authenticated route gating; event APIs require Bearer authentication
**Suggested Layer**: E2E

### TC-212: Created event ownership comes from the authenticated user
**Category**: Security
**Priority**: P0
**Preconditions**: User A and User B have separate valid JWTs
**Steps**:
1. Create an event as User A and record its `userId`
2. Create another event as User B with the same payload
3. Fetch both events as each user
**Expected Results**: Each event is owned by the authenticated token's user; request-body ownership fields cannot claim another user's event
**Business Rule**: `eventService.createEvent` sets `userId` from `req.user.userId`
**Suggested Layer**: API

### TC-213: User B cannot discover User A's private event
**Category**: Security
**Priority**: P0
**Preconditions**: User A owns a dynamic event; User B is authenticated
**Steps**:
1. Send `GET /api/events` as User B with search text matching User A's event
2. Send `GET /api/events/:userA_event_id` as User B
**Expected Results**: The event is absent from list/search results and detail returns 404; no private fields are leaked
**Business Rule**: Repository filters dynamic events by `userId`
**Suggested Layer**: API

### TC-214: Client cannot force a created event to become static
**Category**: Security
**Priority**: P1
**Preconditions**: User is authenticated
**Steps**:
1. Send `POST /api/events` with valid fields plus `isStatic: true`, `userId: null`, and a different `userId`
2. Inspect the response and subsequent list visibility
**Expected Results**: Event is still `isStatic: false`, belongs to the authenticated user, and is private; client ownership flags are ignored
**Business Rule**: `createEvent` constructs a whitelist payload with server-owned `userId` and `isStatic`
**Suggested Layer**: API

## Negative / Error

### TC-315: Create Event form blocks submission when required fields are blank
**Category**: Negative
**Priority**: P0
**Preconditions**: User is logged in; `/admin/events` is open
**Steps**:
1. Leave title, venue, city, date, price, or seats blank one at a time
2. Click `#add-event-btn`
3. Inspect validation text and the Network panel
**Expected Results**: Field-level errors identify the missing value and no `POST /api/events` request is sent; category defaults to Conference
**Business Rule**: `EventForm.validate` required-field checks
**Suggested Layer**: E2E / Component

### TC-316: Create Event API rejects missing required fields
**Category**: Negative
**Priority**: P0
**Preconditions**: User has a valid JWT
**Steps**:
1. Send `POST /api/events` with one or more required fields omitted
2. Inspect the response
**Expected Results**: HTTP 400 with `error: "Validation failed"` and `details` naming each invalid field; no partial event is created
**Business Rule**: `validateCreateEvent` required validators
**Suggested Layer**: API

### TC-317: Past or current event date is rejected by UI and API
**Category**: Negative
**Priority**: P0
**Preconditions**: User is logged in; a valid payload except for date is available
**Steps**:
1. Enter a past or current value in `getByLabel('Event Date & Time')` and submit
2. Send the same value as `eventDate` to `POST /api/events`
**Expected Results**: UI shows `Must be a future date` and does not post; API returns 400 with `Event date must be in the future`
**Business Rule**: Event date must be strictly future-dated
**Suggested Layer**: E2E / API

### TC-318: Create Event API rejects malformed event dates
**Category**: Negative
**Priority**: P1
**Preconditions**: User has a valid JWT
**Steps**:
1. Send valid fields except `eventDate: "tomorrow"` and then `eventDate: ""`
2. Inspect validation details
**Expected Results**: HTTP 400 identifies `eventDate`; malformed values return the ISO 8601 error and blank values return the required-date error
**Business Rule**: `isISO8601()` and required date validator
**Suggested Layer**: API

### TC-319: Create Event rejects negative price and non-positive seats
**Category**: Negative
**Priority**: P0
**Preconditions**: User is authenticated; all other fields are valid
**Steps**:
1. Submit price `-0.01`, then seats `0` and `-1`
2. Repeat the invalid values through `POST /api/events`
**Expected Results**: UI blocks each request with the relevant field error; API returns 400 with the price or seats validation message; no event is created
**Business Rule**: Price must be >= 0 and total seats must be an integer >= 1
**Suggested Layer**: E2E / API

### TC-320: Create Event rejects invalid image URLs at the API boundary
**Category**: Negative
**Priority**: P2
**Preconditions**: User is authenticated; all required values are valid
**Steps**:
1. Send `imageUrl: "not-a-url"` through `POST /api/events`
2. Enter the same value in `getByLabel('Image URL (optional)')` and submit the UI form
**Expected Results**: API returns 400 with `Image URL must be a valid URL`; **source discrepancy**: the form has no client-side URL validation, so it posts the value and displays the API error toast
**Business Rule**: Server validates optional image URL; `EventForm.validate` omits this check
**Suggested Layer**: API / E2E

### TC-321: Create Event API rejects non-integer seats and non-numeric price
**Category**: Negative
**Priority**: P1
**Preconditions**: User has a valid JWT
**Steps**:
1. Send `price: "free"` and then `totalSeats: "2.5"` with otherwise valid fields
2. Inspect validation details
**Expected Results**: HTTP 400 identifies the invalid numeric field; no event is created for either request
**Business Rule**: `isFloat({ min: 0 })` for price and `isInt({ min: 1 })` for total seats
**Suggested Layer**: API

### TC-322: Whitespace-only required text is rejected
**Category**: Negative
**Priority**: P1
**Preconditions**: User is authenticated
**Steps**:
1. Enter spaces into title, category, venue, and city in separate submissions
2. Submit the form and repeat with API payloads containing whitespace-only values
**Expected Results**: UI shows required errors and sends no request; API trims these fields and returns HTTP 400
**Business Rule**: Required text validators use `.trim().notEmpty()`; the form uses `.trim()` checks
**Suggested Layer**: E2E / API

## Edge Cases

### TC-409: Zero-price event is accepted
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User is authenticated; all other fields are valid
**Steps**:
1. Set `getByLabel('Price ($)')` to `0`
2. Create the event and fetch it via `GET /api/events/:id`
**Expected Results**: Creation succeeds and stored price is 0
**Business Rule**: Price lower boundary is 0
**Suggested Layer**: E2E / API

### TC-410: One-seat event is accepted and fully available
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User is authenticated; valid future date and price are available
**Steps**:
1. Set `getByLabel('Total Seats')` to `1`
2. Create the event and fetch it
**Expected Results**: Creation succeeds with `totalSeats: 1` and `availableSeats: 1`; one booking can exhaust its personal availability
**Business Rule**: Total seats lower boundary is 1
**Suggested Layer**: E2E / API

### TC-411: Decimal price precision is preserved
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User is authenticated; all fields are valid
**Steps**:
1. Create an event with price `0.01` and total seats `1`
2. Inspect the API response and detail representation
**Expected Results**: Creation succeeds and price is represented as Decimal value 0.01, without integer rounding
**Business Rule**: Event price is Decimal(10,2); form uses `step="0.01"`
**Suggested Layer**: API

### TC-412: Large valid integer seat count is accepted
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User is authenticated; database accepts the chosen integer
**Steps**:
1. Create an event with `totalSeats: 2147483647`
2. Inspect the response
**Expected Results**: Validation accepts the positive integer and `availableSeats` equals it; the create flow does not silently truncate it
**Business Rule**: Validator requires only a positive integer; no business maximum is defined
**Suggested Layer**: API

### TC-413: Sixth-to-seventh transition prunes only dynamic events
**Category**: Edge Case
**Priority**: P0
**Preconditions**: User has six dynamic events and seeded static events are visible
**Steps**:
1. Record IDs of all six dynamic events and one static event
2. Create one more dynamic event
3. Fetch `GET /api/events?limit=100`
**Expected Results**: Exactly the oldest dynamic event is removed; static events remain untouched and dynamic count is six
**Business Rule**: FIFO applies only to `userId` + `isStatic: false`
**Suggested Layer**: API

### TC-414: Duplicate event content is allowed
**Category**: Edge Case
**Priority**: P2
**Preconditions**: User has fewer than six dynamic events
**Steps**:
1. Submit the same valid payload twice
2. Compare both responses and list results
**Expected Results**: Both requests succeed with distinct event IDs; no uniqueness rule exists for event content
**Business Rule**: Event model has no uniqueness constraint on title, date, venue, or complete payload
**Suggested Layer**: API

### TC-415: Arbitrary non-empty category and city are accepted by the current API
**Category**: Edge Case
**Priority**: P1
**Preconditions**: User is authenticated; other fields are valid
**Steps**:
1. Send `category: "Opera"` and `city: "Kolkata"` to `POST /api/events`
2. Fetch the event and inspect `/events`
**Expected Results**: **Actual current behavior**: API returns 201 and preserves both strings because validation only checks non-empty values. This conflicts with the documented category/city sets; the UI cannot select arbitrary categories because `EventForm` provides five fixed options, while city is free text
**Business Rule**: Domain lists supported values; **source discrepancy**: server does not enforce those lists
**Suggested Layer**: API / E2E

## UI State

### TC-511: Admin Event list shows a spinner while events are fetched
**Category**: UI State
**Priority**: P2
**Preconditions**: User is authenticated; `/admin/events` is opened on a slow connection
**Steps**:
1. Throttle `GET /api/events`
2. Observe the All Events section before the response arrives
**Expected Results**: A large `Spinner` is shown; the table and empty state are not rendered until loading completes; the create form remains available
**Business Rule**: `AdminEventsPage` `isLoading` branch
**Suggested Layer**: Component / E2E

### TC-512: Admin Event list shows empty state when no events are visible
**Category**: UI State
**Priority**: P1
**Preconditions**: User is logged in and the events response is empty
**Steps**:
1. Navigate to `/admin/events`
2. Observe the All Events section
**Expected Results**: `EmptyState` shows `No events yet` and `Create your first event using the form above.` while the form remains usable
**Business Rule**: `events.length === 0` branch in `AdminEventsPage`
**Suggested Layer**: E2E / Component

### TC-513: Create Event validation preserves entered values
**Category**: UI State
**Priority**: P1
**Preconditions**: User is on `/admin/events`
**Steps**:
1. Enter valid title, venue, city, and date but leave price invalid
2. Click `#add-event-btn`
3. Correct only the price and inspect the other fields before resubmitting
**Expected Results**: Price error is shown; entered values remain populated; after correction the form can submit without re-entering other values
**Business Rule**: `setErrors` changes validation state without resetting `form`
**Suggested Layer**: Component / E2E

### TC-514: Successful creation resets the form and shows feedback
**Category**: UI State
**Priority**: P0
**Preconditions**: User is authenticated; valid creation payload is ready
**Steps**:
1. Submit `#admin-event-form`
2. Observe the toast and all form controls after the 201 response
**Expected Results**: Toast says `Event created!`; all fields reset to `EMPTY` defaults, category returns to Conference, and the new row appears after query invalidation
**Business Rule**: `useCreateEvent` success callback in `EventForm`
**Suggested Layer**: E2E / Component

### TC-515: Submit button indicates an in-flight create request
**Category**: UI State
**Priority**: P1
**Preconditions**: User has filled a valid form; `POST /api/events` is delayed
**Steps**:
1. Click `#add-event-btn`
2. Observe the button while the request is pending
3. Attempt a second submission
**Expected Results**: Button enters its loading state and is disabled by the shared Button component; duplicate submissions are prevented while `creating` is true
**Business Rule**: `pending = creating || updating` and Button loading behavior
**Suggested Layer**: Component / E2E

### TC-516: API create failure displays an error toast and preserves the form
**Category**: UI State
**Priority**: P1
**Preconditions**: User is authenticated; server returns 400 or 500 for the create request
**Steps**:
1. Fill a form that produces a server-side error, such as an invalid image URL
2. Click `#add-event-btn`
3. Observe the toast and form values
**Expected Results**: Error toast displays the API message; form is not reset; no new row appears; submit control returns to idle
**Business Rule**: `onError` calls `toast(err.message, 'error')`; reset occurs only in `onSuccess`
**Suggested Layer**: E2E / Component

### TC-517: Admin table marks static events read-only
**Category**: UI State
**Priority**: P1
**Preconditions**: At least one seeded static event is returned by `GET /api/events`
**Steps**:
1. Navigate to `/admin/events`
2. Locate the static row by its `Featured` marker
3. Inspect its Actions cell
**Expected Results**: Static row shows `Read-only`; edit and delete controls are absent for that row, while dynamic rows expose them
**Business Rule**: Static events are immutable and cannot be edited or deleted
**Suggested Layer**: E2E

### TC-518: Sandbox warning reflects the source thresholds
**Category**: UI State
**Priority**: P2
**Preconditions**: Events query returns more than five visible events
**Steps**:
1. Navigate to `/admin/events` and `/events`
2. Observe the warning banners
**Expected Results**: Admin page always warns that six events are allowed; public page shows its sandbox banner when `events.length > 5`. **Source discrepancy**: the domain describes a near-six threshold, while the public source uses strictly greater than five and the admin source always renders its warning
**Business Rule**: Sandbox warning and six-event FIFO limit
**Suggested Layer**: E2E / Component
