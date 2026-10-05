# Smart Slot Booking Product Requirements Document
A clean-room product requirements document for the Smart Slot Booking faculty office hours engine.

## Problem
200 students book faculty office hours and lab slots simultaneously, leading to scheduling friction, timezone errors, and calendar clashes.
The system must guarantee zero double bookings across time zones with buffer times and day-off overrides.

## Target Users
- Faculty Host: Sets weekly availability in local timezone, marks day-off overrides, and conducts student office hours.
- Student Booker: Browses open slots converted to local timezone, reserves a slot, and confirms meeting appointments.
- System Administrator: Monitors scheduling health, database constraints, and booking audit records.

## Core User Flows

### 1. Host Defines Availability and Overrides
1. Host selects weekly available days and time intervals in host IANA timezone.
2. Host adds date overrides to block holidays or entire days off.
3. System stores recurring rules and overrides with host timezone identifier.

### 2. Booker Discovers Available Slots
1. Student accesses faculty booking link and supplies student IANA timezone.
2. Server expands host recurring schedule into UTC intervals for requested date window.
3. Server subtracts day-off overrides, existing bookings, and buffer times.
4. Server slices open windows into discrete slots and converts timestamps to student timezone.
5. Student views open, selectable time slots.

### 3. Booker Reserves and Confirms Slot
1. Student selects an open slot, creating a temporary reservation hold.
2. Student inputs name and email address.
3. Student submits booking request with reservation token.
4. Server recomputes availability and verifies conflict-free status in database transaction.
5. Database exclusion constraint prevents concurrent collision.
6. Server releases reservation hold and creates confirmed booking record.

### 4. Cancellation Flow
1. Host or student submits cancellation request with booking identifier.
2. Server updates booking status to CANCELLED.
3. Released slot immediately reappears in availability queries.

## Scope

### Must Have
- Instants stored strictly in UTC with IANA timezone strings.
- Timezone translation between host and student timezones.
- Half-open time intervals `[start, end)` for all slots and bookings.
- Pre-event and post-event buffer times expanding blocked intervals.
- Day-off overrides evaluated in host timezone.
- Server-side availability recomputation on every booking attempt.
- Database-level exclusion constraint preventing concurrent double bookings.
- Stable error code `SLOT_ALREADY_BOOKED` with HTTP 409 on conflict.

### Should Have
- Expiring reservation tokens in `SelectedSlots` table.
- Email confirmation notifications for host and student.
- Self-serve booking cancellation endpoint.

### Could Have
- External Google Calendar two-way busy time sync.
- Configurable minimum notice window before booking start.

### Won't Have (This Release)
- Multi-host team round-robin routing.
- Paid bookings and Stripe payment integration.
- Recurring booking series creation.
- Arbitrary custom questionnaire builders.

## Out of Scope
- Single Sign-On (SAML/Okta) enterprise federation.
- SMS and WhatsApp reminder notifications.
- Native mobile applications for iOS or Android.

## Acceptance Criteria

### KT1: Time Zone Support (Host in IST, Booker in PST)
- Given a host in `Asia/Kolkata` (IST, UTC+5:30) with availability 10:00 to 11:00 IST on Thursday, October 15, 2026.
- And event length is 30 minutes with 0 minutes buffer time.
- And a student booker is in `America/Los_Angeles` (PDT, UTC-7:00).
- When student requests open slots for host on October 15, 2026 IST.
- Then host sees slots: `10:00 - 10:30 IST` and `10:30 - 11:00 IST` on 2026-10-15.
- And student sees slots: `21:30 - 22:00 PDT` and `22:00 - 22:30 PDT` on Wednesday, October 14, 2026.
- And stored UTC instants are `2026-10-15T04:30:00Z` and `2026-10-15T05:00:00Z`.

### KT2: Buffer Time Enforcement
- Given a host in `Asia/Kolkata` available 14:00 to 16:00 IST on 2026-10-15 (`08:30Z - 10:30Z`).
- And event duration is 30 minutes with `before_buffer = 15` minutes and `after_buffer = 15` minutes.
- And an existing booking exists from 14:30 to 15:00 IST (`09:00Z - 09:30Z`).
- And blocked interval formula `[start - before_buffer, end + after_buffer)` blocks `[14:15, 15:15) IST` (`[08:45Z, 09:45Z)`).
- When a student queries available slots on 30-minute start intervals.
- Then 14:00 slot is blocked because `[13:45, 14:45)` overlaps `[14:15, 15:15)`.
- And 14:30 slot is blocked by the existing meeting.
- And 15:00 slot is blocked because `[14:45, 15:45)` overlaps `[14:15, 15:15)`.
- And only 15:30 slot (`[15:15, 16:15)`) is returned as available.

### KT3: Double Booking Prevention under Concurrency
- Given an open slot at 10:00 - 10:30 IST on 2026-10-15 (`04:30Z - 05:00Z`) with 0 buffer minutes.
- And Student A and Student B both submit booking requests concurrently for `2026-10-15T04:30:00Z`.
- When both requests execute inside database transactions.
- Then exactly one transaction succeeds, inserts the booking record, and returns HTTP 201 Created.
- And the competing transaction violates the PostgreSQL exclusion constraint and rolls back.
- And the losing request receives HTTP 409 Conflict with error code `SLOT_ALREADY_BOOKED`.
- And the database contains exactly one booking record for that slot.

### Day-Off Override
- Given a host in `Asia/Kolkata` with recurring Thursday availability 10:00 to 12:00 IST.
- And host records a day-off override on date `2026-10-15` in `Asia/Kolkata`.
- When student requests open slots for 2026-10-15.
- Then server evaluates override in host timezone and returns zero available slots.

### Basic Successful Booking
- Given an open slot 11:00 to 11:30 IST on 2026-10-16 with no conflicts.
- When student submits valid name and email with an active reservation token.
- Then server inserts booking record, marks status CONFIRMED, and returns HTTP 201 Created.

### Conflicting Booking Rejected
- Given an existing confirmed booking 11:00 to 11:30 IST on 2026-10-16.
- When another student submits booking request for 11:00 to 11:30 IST without an active hold.
- Then server availability check detects conflict and returns HTTP 409 Conflict with code `SLOT_ALREADY_BOOKED`.
