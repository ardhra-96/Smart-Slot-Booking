# Smart Slot Booking — HACKBACK Submission

## Metadata
- **Card**: DBG-922 · BAKLAVA
- **Original**: Cal.com
- **Rebuild Scope**: Clean-room overnight rebuild of the faculty office hours and lab slot scheduling engine.

---

## Brief Implementation Summary
Built completely from scratch using Next.js 16 (App Router), TypeScript, Vanilla CSS, and PostgreSQL 16 managed via Prisma ORM:
- **Zero Cal.com code reuse**: Built strictly from specifications in `/docs`.
- **Database-level concurrency protection**: Solved the critical Cal.com race condition by adding a PostgreSQL `btree_gist` GiST exclusion constraint (`no_overlapping_bookings`) that mathematically eliminates overlapping meetings for the same host.
- **Precision timezone translation**: Evaluates host schedules in local wall-clock time (`Asia/Kolkata`) and translates discrete intervals to the booker's timezone (`America/Los_Angeles`) across calendar day boundaries without date drift.
- **Half-open buffer arithmetic**: Pre-event and post-event buffer times are applied as half-open ranges `[start - beforeBuffer, end + afterBuffer)` to prevent back-to-back scheduling collisions.
- **Atomic reservation holds**: Temporary reservation tokens placed in `SelectedSlots` are verified and atomically consumed within database transactions.

---

## The 3 Mandatory Killer Tests

### 1. Killer Test 1: Time Zone Support (Host in IST, Booker in PST)
- **Scenario**: Host in `Asia/Kolkata` (IST, UTC+5:30) available `10:00 - 11:00 IST` on Thursday, October 15, 2026 for a 30-minute event with 0 buffer. Student booker is in `America/Los_Angeles` (PDT, UTC-7).
- **Result**:
  - Host queries October 15, 2026 IST and sees slots: `10:00 - 10:30 IST` and `10:30 - 11:00 IST` on `2026-10-15`.
  - Student queries October 14–15 and sees slots: `21:30 - 22:00 PDT` and `22:00 - 22:30 PDT` on Wednesday, October 14, 2026.
  - Stored database UTC timestamps are `2026-10-15T04:30:00.000Z` and `2026-10-15T05:00:00.000Z`.
- **Status**: **PASS** (verified in `tests/killer-tests.test.ts` and `tests/run-killer-tests.ts`).

### 2. Killer Test 2: Buffer Time Enforcement
- **Scenario**: Host in `Asia/Kolkata` available `14:00 - 16:00 IST` on `2026-10-15` with `before_buffer = 15` and `after_buffer = 15` minutes. An existing booking is confirmed from `14:30 - 15:00 IST`.
- **Blocked Interval**: `[14:15, 15:15) IST` (`[08:45Z, 09:45Z)`).
- **Result**:
  - `14:00` slot is blocked (`[13:45, 14:45)` overlaps `[14:15, 15:15)`).
  - `14:30` slot is blocked by the meeting.
  - `15:00` slot is blocked (`[14:45, 15:45)` overlaps `[14:15, 15:15)`).
  - Exactly ONE slot is returned: `15:30 IST` (`[15:15, 16:15)`).
- **Status**: **PASS** (verified in `tests/killer-tests.test.ts` and `tests/run-killer-tests.ts`).

### 3. Killer Test 3: Double Booking Prevention under Concurrency
- **Scenario**: Two concurrent students (Student A and Student B) simultaneously submit booking requests for the same open slot `2026-10-15T04:30:00Z` (10:00 IST).
- **Result**:
  - Both requests enter database transactions.
  - Exactly one transaction commits and returns HTTP `201 Created` with status `CONFIRMED`.
  - Competing transaction violates PostgreSQL exclusion constraint (`no_overlapping_bookings`), rolls back, and returns HTTP `409 Conflict` with code `SLOT_ALREADY_BOOKED`.
  - Exactly one booking record exists in the database.
- **Status**: **PASS** (verified in `tests/killer-tests.test.ts` and `tests/run-killer-tests.ts`).

---

## GAPS Improvements Implemented

1. **Improvement 1: PostgreSQL GiST Exclusion Constraint (`no_overlapping_bookings`)**
   - **Problem in Cal.com**: Cal.com relied purely on application-level in-memory conflict checks (`ensureAvailableUsers`). Lacking database-level mutual exclusion or range constraints on `Booking`, concurrent racing requests double-booked identical host intervals.
   - **Rebuild Fix**: Enabled `btree_gist` extension in PostgreSQL and added a GiST exclusion constraint on host ID and buffered intervals:
     ```sql
     ALTER TABLE "Booking" ADD CONSTRAINT "no_overlapping_bookings"
     EXCLUDE USING gist (
       "userId" WITH =,
       booking_blocked_range("startTime", "endTime", "beforeBuffer", "afterBuffer") WITH &&
     ) WHERE ("status" != 'CANCELLED');
     ```
     Any simultaneous colliding transaction is stopped at the database engine level with SQLSTATE `23P01`, triggering automatic rollback and HTTP 409 `SLOT_ALREADY_BOOKED`.

2. **Improvement 2: Atomic Reservation Token Validation and Hold Release**
   - **Problem in Cal.com**: Temporary holds were saved in `SelectedSlots` but completely ignored during booking creation in `RegularBookingService`, allowing racing API calls to steal held slots from students filling out checkout forms.
   - **Rebuild Fix**: Implemented atomic verification and deletion of the reservation hold inside the booking database transaction. Unheld slots cannot steal an active hold, and the hold is atomically consumed upon confirmation.

---

---

## Unique Features: Smart Slot Recommendation & Explainable Slot Insights

### 1. Smart Slot Recommendation
- **Behavior**:
  - When a student browses available dates, they can specify an optional preferred booking time (e.g. morning, afternoon, evening, or specific hour `HH:mm`).
  - The algorithm calculates the absolute distance between each available slot's start time and the student's target preference in the student's local timezone.
  - The top 3 closest matches are displayed as highlighted "✨ Smart Recommended" options with ranking badges (`⭐ Best Match`, `Pick #2`, `Pick #3`).
  - If no preference is provided, the engine defaults to recommending the earliest 3 available slots of the day.
- **Deterministic & Zero External Dependencies**: Runs entirely locally via deterministic mathematical ranking; requires zero AI API keys or third-party cloud services.

### 2. Explainable Slot Insights
**Explainable Slot Insights — the system does not merely show available slots; it explains why a slot is recommended and, where appropriate, why a slot is unavailable.**

- **Available Slot Explanations**: Concise context for every slot (e.g. "Best match for your preferred time", "Closest available slot to your preference", "Earliest available slot", "Fits your selected morning/afternoon/evening preference").
- **Timezone Explanations**: Dual-timezone transparency showing host local time and booker local time (e.g. "Host time: 10:00 AM IST · Your time: 9:30 PM PDT") using the backend scheduling engine as the single source of truth.
- **Buffer Explanations**: When a slot is blocked by adjacent meetings and pre/post buffers, presents a clean, user-friendly reason: `"Unavailable because it overlaps the host's required buffer time."` without exposing internal database errors.
- **Day-Off Explanations**: When faculty marks a date as off via schedule overrides, clearly communicates: `"Faculty is unavailable on this date."`
- **Recommendation Reasons**: Transparent ranking explanations on Smart Recommended slots (e.g., `"Best match — 10 minutes from your preferred time"`, `"Pick #2 — 25 minutes from your preferred time"`, `"Earliest available slot"`).
- **Clean UI**: Integrated "Why this slot?" expandable cards and buffer explanation toggles preserving the sleek dark-mode aesthetic.
- **100% Deterministic**: Calculated purely from scheduling intervals and time offsets without AI or external APIs.

---

## Automated Test Verification Summary

```text
✓ tests/killer-tests.test.ts (11 tests) 117ms
   ✓ Smart Slot Booking — Killer Tests & Core Verifications
     ✓ Killer Test 1: A host in IST and a booker in PST both see the correct slots
     ✓ Killer Test 2: Buffer time between bookings is respected
     ✓ Killer Test 3: Two simultaneous bookings for the same slot result in exactly one success
     ✓ Day-off override in host timezone completely removes available slots
     ✓ Cancellation frees slot immediately
     ✓ GAPS Improvement 2: Atomic reservation hold prevents race conditions and is consumed upon booking
     ✓ Differentiator: Smart Slot Recommendation ranks available slots by student preferred time or defaults to earliest 3
     ✓ Explainable Slot Insights: recommendation reason explains ranking distance to preferred time or earliest
     ✓ Explainable Slot Insights: timezone display and explanation clearly shows host time and booker local time
     ✓ Explainable Slot Insights: buffer conflict explanation exposes user-friendly reason for buffer-protected slots
     ✓ Explainable Slot Insights: day-off explanation indicates faculty unavailability on date override

Test Files  1 passed (1)
Tests       11 passed (11)
```
