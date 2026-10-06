# Smart Slot Booking

A clean-room faculty office hours and lab slot scheduling platform built to eliminate double bookings, resolve cross-timezone scheduling friction, and enforce pre/post meeting buffer times under high concurrency.

---

## Quickstart (4 Commands)

Run the following commands to install dependencies, migrate the schema, run tests, and start the development server:

```bash
# 1. Install dependencies
npm install

# 2. Sync database schema and seed demonstration data
npx prisma db push && npm run seed

# 3. Run automated Killer Tests suite
npm test

# 4. Launch local development server
npm run dev
```

Visit `http://localhost:3000` to browse available faculty office hours, or `http://localhost:3000/prof.sharma/office-hours` to book a slot directly.

---

## Problem & Solution

When hundreds of students attempt to book faculty office hours simultaneously across different time zones, legacy scheduling systems encounter race conditions, double-booked slots, calendar drift, and missing buffer boundaries.

**Smart Slot Booking solves this through:**
1. **Database-Level Mutual Exclusion**: PostgreSQL `btree_gist` exclusion constraint (`no_overlapping_bookings`) guaranteeing that overlapping meeting intervals for the same host are rejected at the database engine level (`409 Conflict` with `SLOT_ALREADY_BOOKED`).
2. **Deterministic Timezone Projection**: All instants are stored strictly in UTC ISO-8601 (`timestamptz`). Host recurring rules and date overrides are evaluated in host wall-clock time (`Asia/Kolkata`) and translated to student local time (`America/Los_Angeles`) across calendar day boundaries without drift.
3. **Half-Open Buffer Enforcement**: Meeting intervals and pre/post buffers are evaluated as half-open ranges `[start - beforeBuffer, end + afterBuffer)` to prevent back-to-back overlaps.
4. **Atomic Reservation Holds**: Short-lived hold tokens in `SelectedSlots` protect slots during checkout and are atomically validated and consumed within the booking transaction.
5. **Smart Slot Recommendation**: Deterministically ranks available slots based on the student's preferred booking time (e.g. morning, afternoon, or specific hour) and displays the best 3 recommendations first, functioning seamlessly without any external AI API key.
6. **Explainable Slot Insights**: The system explains why a slot is recommended, available, or unavailable (including buffer conflict explanations, faculty day-off overrides, and cross-timezone dual perspectives) with integrated "Why this slot?" expandable cards.

---

## The 3 Mandatory Killer Tests

All 3 Killer Tests are implemented and verified via automated test suites (`tests/killer-tests.test.ts` and `tests/run-killer-tests.ts`):

1. **KT1: Time Zone Support (Host in IST, Booker in PST)**
   - Host in `Asia/Kolkata` available 10:00 to 11:00 IST on Thursday, October 15, 2026.
   - Host sees slots: `10:00 - 10:30 IST` and `10:30 - 11:00 IST` on `2026-10-15`.
   - Booker in `America/Los_Angeles` sees slots: `21:30 - 22:00 PDT` and `22:00 - 22:30 PDT` on Wednesday, `2026-10-14`.
   - Stored UTC instants: `2026-10-15T04:30:00Z` and `2026-10-15T05:00:00Z`.

2. **KT2: Buffer Time Enforcement**
   - Host in `Asia/Kolkata` available 14:00 to 16:00 IST on `2026-10-15` with 15m pre-buffer and 15m post-buffer.
   - Existing confirmed booking `14:30 - 15:00 IST` blocks interval `[14:15, 15:15) IST`.
   - 14:00 slot is blocked (`[13:45, 14:45)` overlaps `[14:15, 15:15)`).
   - 14:30 slot is blocked by the meeting.
   - 15:00 slot is blocked (`[14:45, 15:45)` overlaps `[14:15, 15:15)`).
   - Only `15:30 IST` (`[15:15, 16:15)`) is returned as available.

3. **KT3: Double Booking Prevention under Concurrency**
   - Concurrent submission of 2 simultaneous booking requests for the same open slot (`2026-10-15T04:30:00Z`).
   - Exactly one transaction commits and returns HTTP `201 Created` (`CONFIRMED`).
   - Competing transaction triggers PostgreSQL `exclusion_violation` (SQLSTATE `23P01`), rolls back, and returns HTTP `409 Conflict` with `SLOT_ALREADY_BOOKED`.
   - Exactly one booking record exists in the database.

---

## API Endpoints

| Method | Path | Description | Status |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/slots` | Returns available and recommended slots normalized to booker timezone | `200 OK` |
| `POST` | `/api/reserve` | Places temporary 5-minute reservation hold in `SelectedSlots` | `201 Created` |
| `POST` | `/api/bookings` | Confirms booking atomically with GiST exclusion protection | `201 Created` / `409 Conflict` |
| `DELETE` | `/api/bookings/:id` | Cancels confirmed booking by UID and frees slot | `200 OK` |
| `GET/POST` | `/api/admin/schedule` | Manages host weekly recurring rules and date overrides | `200 OK` / `201 Created` |

---

## Tech Stack
- **Framework**: Next.js 16 (App Router, TypeScript)
- **Styling**: Vanilla CSS (Tailwind-free, responsive dark-mode design system)
- **Database**: PostgreSQL 16 with `btree_gist` extension
- **ORM**: Prisma Client 6.4.1
- **Testing**: Vitest 5 & tsx runner
