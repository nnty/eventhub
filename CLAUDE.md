# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

EventHub is a full-stack event ticket booking platform built for QA training/test-automation practice. Users register, browse events, book tickets, and manage bookings. Each user's data is an isolated sandbox layered on top of shared seed data (see "Per-user sandbox model" below).

## Tech Stack

- **Frontend**: Next.js 14 (App Router), React 18, TypeScript/JS mixed, Tailwind CSS, React Query v5
- **Backend**: Express.js, Prisma ORM, MySQL 8+
- **Auth**: JWT (7-day expiry, `Bearer` header), bcryptjs
- **Testing**: Playwright (Chromium only), tests run against a **hosted deployment**, not localhost, by default

## Commands

```bash
npm run setup    # install deps in both backend/ and frontend/
npm run dev      # start backend (3001) + frontend (3000) concurrently
npm run db:push  # push Prisma schema to DB, no migration files (non-interactive)
npm run migrate  # prisma migrate dev — interactive, creates migration files
npm run seed     # seed 3 static events + 2 test users (backend/prisma/seed.js)
npm run build    # next build (frontend)
npm run lint     # next lint (frontend only; backend has no lint script)
npm run test     # run all Playwright tests
npm run test:ui  # Playwright UI mode
npx playwright test tests/booking-management.spec.js --reporter=line  # single test file
```

Backend-only commands (run from `backend/`): `npm run prisma:generate`, `npm run prisma:studio`.

There is no automated test runner for the backend or frontend beyond `tsc --noEmit` (frontend type-check, run via CI) and `node --check` (backend syntax check, run via CI) — see `.github/workflows/ci.yml`.

## Architecture

Backend is layered: `routes/` → `controllers/` → `services/` → `repositories/` (Prisma) → MySQL. Controllers are thin (parse req, call service, `next(err)`); business logic and validation live in `services/`; `repositories/` are pure Prisma data access. Domain errors (`NotFoundError`, `ForbiddenError`, `InsufficientSeatsError`, `ValidationError` in `backend/src/utils/errors.js`) are thrown from services and mapped to HTTP responses centrally in `backend/src/middleware/errorHandler.js` (also maps Prisma `P2002`/`P2025`/`P2003`).

### Per-user sandbox model

This is the key non-obvious concept in the codebase. `Event.availableSeats` in the DB is a **shared** static/global value, but every API response recomputes a **personal** available-seat count per requesting user by subtracting that user's own booked quantity for the event (`eventService.withPersonalSeats`, `bookingService.createBooking`). This means:
- Two different users see different "available seats" for the same event.
- Cancelling a booking does *not* restore the DB's `availableSeats` for dynamic events — it doesn't need to, since availability is computed live from that user's bookings.
- Static (seeded) events (`isStatic: true`, `userId: null`) are shared/immutable and visible to all users; user-created events are private to their creator.

### FIFO limits

- Max 6 user-created (dynamic) events per user — oldest is deleted on overflow (`eventService.createEvent`).
- Max 9 bookings per user — oldest is deleted on overflow, preferring to evict a booking for a *different* event than the one being booked; if the fallback evicts a booking for the *same* event, a seat is permanently burned via `eventRepository.decrementSeats` so the personal count still reflects it (`bookingService.createBooking`).

### Other business rules

- Booking reference format: `<FirstLetterOfEventTitle>-XXXXXX` (uppercase), generated with collision retry (`bookingService.generateUniqueRef`).
- Refund eligibility is **client-side only, not enforced by the API**: exactly 1 ticket = eligible, >1 = not (`frontend/app/bookings/[id]/page.tsx`, `RefundEligibility` component).
- Cross-user access to another user's booking/event returns 403 via `ForbiddenError`, not 404.
- Static events cannot be updated or deleted (`ForbiddenError`).

### Auth

JWT-based; `authMiddleware.js` verifies the `Authorization: Bearer <token>` header and sets `req.user = { userId, email }`. Frontend stores the token in `localStorage` (`eventhub_token`) via `AuthProvider` (`frontend/lib/hooks/useAuth.tsx`) and gates routes with `AuthGuard` (`frontend/components/auth/AuthGuard.tsx`), which redirects unauthenticated users to `/login` unless the path is in `PUBLIC_PATHS`.

### Frontend API layer — duplicate files, only one set is live

`frontend/lib/api/` contains **two parallel implementations**: a `.ts` set (`client.ts`, `events.ts`, `bookings.ts`, `index.ts`) and a `.js` set (`client.js`, `authApi.js`, `eventsApi.js`, `bookingsApi.js`). Only the `.js` set is actually imported anywhere in the app (`useAuth.tsx` → `authApi.js`, `useEvents.ts`/`useBookings.ts` → `eventsApi.js`/`bookingsApi.js`, pages → `client.js` for `BASE_URL`). The `.ts` set is dead code — don't edit it expecting it to take effect, and don't assume `axios` is used (the README describes an older axios-based client; the live code uses `fetch`).

## Testing

Playwright's `baseURL` in `playwright.config.ts` is hardcoded to `https://eventhub.rahulshettyacademy.com` — a hosted deployment, not `http://localhost:3000`. Running `npm run test` locally hits the live hosted site by default; there is no env-var override wired up, so testing against a local dev server requires temporarily editing `playwright.config.ts`. CI's `playwright.yml` runs the same suite against that same hosted URL on every push to `main`.

- Test files: `tests/<feature-name>.spec.js`
- Test account (seeded, also used by the one existing spec): `rahulshetty1@gmail.com` / `Magiclife1!` (a second seeded account: `rahulshetty1@yahoo.com` / same password)
- Elements are targeted via `data-testid` attributes (see `tests/booking-management.spec.js` for the established pattern of small `login()`/`bookEvent()`/`clearBookings()` helpers reused across tests)
- No `page.waitForTimeout()` — use `expect().toBeVisible()`
- Tests must be self-contained (login → clear state → action → assert)

## CI/CD

- `ci.yml`: PR gate — backend syntax check + Prisma validate/format/generate (no DB), frontend `tsc --noEmit` + production build. Also runs a schema-drift job that SSHes into the production server to diff the PR's Prisma schema against the live DB (read-only).
- `playwright.yml`: runs the full Playwright suite against the hosted production site on every push to `main`.
- `deploy.yml`: on push to `main`, reuses `ci.yml` as a gate, then SSHes into the production server, pulls `main`, runs `prisma generate` (not `migrate deploy`), rebuilds the frontend, and reloads PM2. **Prisma migrations are never run automatically** — if a change includes a migration, it must be applied manually on the server (`npx prisma migrate deploy`) before triggering deploy.
