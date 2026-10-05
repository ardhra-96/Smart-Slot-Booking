# Smart Slot Booking API Specification
A clean-room API specification describing the endpoints, procedure contracts, and concurrency workflows for the rebuild.

## Summary of Rebuild Endpoints
| Method | Path | Purpose | Auth | Who may call |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/slots` | Returns available slots normalized to booker timezone | Public | Any student booker |
| `POST` | `/api/reserve` | Holds a selected slot temporarily | Public | Prospective booker |
| `POST` | `/api/bookings` | Confirms and creates a booking | Public | Student booking a slot |
| `DELETE` | `/api/bookings/:id` | Cancels an existing booking appointment | Public / UID | Student or host |

## Rebuild Endpoint Specifications

### 1. GET /api/slots
- Method: `GET`
- Path: `/api/slots`
- Purpose: Computes available time slots for a host across a requested date range in the booker timezone.
- Auth: Public.
- Who may call: Any public student invitee or web client.
- Input: Query parameters: `hostId` (integer), `slug` (string), `startDate` (ISO-8601 string), `endDate` (ISO-8601 string), `timeZone` (IANA string).
- Output: `{ slots: Array<{ time: string, utcTime: string }> }`.
- Validation: Validates IANA timezone name and ensures `endDate > startDate`.
- Errors: 400 Bad Request (invalid date or unknown timezone), 404 Not Found (host or event type missing).

### 2. POST /api/reserve
- Method: `POST`
- Path: `/api/reserve`
- Purpose: Places a temporary reservation hold on an open slot.
- Auth: Public.
- Who may call: Prospective student selecting a time slot in the booking interface.
- Input: JSON payload: `{ hostId: number, eventTypeId: number, slotUtcStartDate: string, slotUtcEndDate: string }`.
- Output: `{ reservationToken: string, releaseAt: string }`.
- Validation: Verifies slot is within host working hours and not booked.
- Errors: 400 Bad Request, 404 Not Found, 409 Conflict with code `SLOT_ALREADY_BOOKED`.

### 3. POST /api/bookings
- Method: `POST`
- Path: `/api/bookings`
- Purpose: Validates availability, consumes reservation token, and inserts confirmed booking.
- Auth: Public.
- Who may call: Student confirming a slot booking.
- Input: JSON payload: `{ eventTypeId: number, hostId: number, start: string, end: string, name: string, email: string, timeZone: string, reservationToken: string }`.
- Output: `{ id: number, uid: string, status: "CONFIRMED", start: string, end: string, attendee: { name: string, email: string } }`.
- Validation: Checks required attendee fields, valid email format, and ISO UTC dates.
- Errors: 400 Bad Request, 404 Not Found, 409 Conflict with code `SLOT_ALREADY_BOOKED`.

### 4. DELETE /api/bookings/:id
- Method: `DELETE`
- Path: `/api/bookings/:id`
- Purpose: Cancels a confirmed booking appointment.
- Auth: Public with booking UID or authenticated host.
- Who may call: Meeting invitee holding booking UID or host.
- Input: Route parameter `id` (booking UID string).
- Output: `{ success: true, status: "CANCELLED" }`.
- Validation: Verifies booking exists and is not already cancelled.
- Errors: 404 Not Found.

## Ordered Create-Booking Execution Steps
1. Handler receives POST payload at `/api/bookings` and parses input schema.
2. Validates attendee name, email address, and IANA timezone format.
3. Recomputes host availability from weekly schedule and checks day-off overrides.
4. Checks half-open interval overlap `[start - beforeBuffer, end + afterBuffer)` against active bookings.
5. Verifies `reservationToken` in `SelectedSlots` matches host, start, and end, and is unexpired.
6. Opens PostgreSQL database transaction.
7. Deletes matching reservation hold row from `SelectedSlots`.
8. Inserts new `Booking` and `Attendee` records into PostgreSQL.
9. Database evaluates GiST exclusion constraint on host ID and blocked interval.
10. Commits transaction and returns HTTP 201 Created with booking confirmation payload.

## Simultaneous Requests Behavior
- Two concurrent booking requests A and B submit identical host and start time parameters.
- Both requests recompute availability and enter database transactions.
- Transaction A inserts first; PostgreSQL exclusion constraint verifies no overlapping active booking exists.
- Transaction A commits, removes its reservation token, and returns HTTP 201 Created with status `CONFIRMED`.
- Transaction B attempts insert; PostgreSQL detects range overlap with Transaction A's blocked window.
- PostgreSQL throws an exclusion violation (`exclusion_violation`, SQLSTATE 23P01).
- Transaction B catches database error, rolls back immediately, and returns HTTP 409 Conflict with code `SLOT_ALREADY_BOOKED`.
- Exactly one booking record exists in the database.

## Reference: Original (Cal.com)
A verified technical specification of Cal.com core scheduling routes, procedure contracts, and execution workflows.

### Summary of Core Endpoints (Original)
| Method | Path or Procedure | Purpose | Auth | Evidence |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/book/event` | Creates an individual booking | Public | apps/web/pages/api/book/event.ts:17-63 [Confirmed] |
| `POST` | `/api/book/recurring-event` | Creates recurring booking instances | Public | apps/web/pages/api/book/recurring-event.ts:29-66 [Confirmed] |
| `query` | `viewer.slots.getSchedule` | Computes available time slots | Public | packages/trpc/server/routers/viewer/slots/_router.tsx:18-25 [Confirmed] |
| `mutation` | `viewer.slots.reserveSlot` | Holds a slot temporarily | Public | packages/trpc/server/routers/viewer/slots/_router.tsx:26-33 [Confirmed] |
| `mutation` | `viewer.slots.removeSelectedSlotMark` | Releases a held slot | Public | packages/trpc/server/routers/viewer/slots/_router.tsx:46-55 [Confirmed] |
| `mutation` | `viewer.availability.schedule.update` | Updates weekly hours and overrides | Authed | packages/trpc/server/routers/viewer/availability/schedule/update.handler.ts:15-22 [Confirmed] |
| `DELETE` | `/api/cancel` | Cancels a confirmed booking | Public / Session | apps/web/app/api/cancel/route.ts:16-70 [Confirmed] |

### Booking Creation Sequence (Original)
1. Handler receives POST request at `/api/book/event` and extracts client IP.
   Evidence: apps/web/pages/api/book/event.ts:18 [Confirmed]
2. Validates Cloudflare Turnstile token if enabled.
   Evidence: apps/web/pages/api/book/event.ts:20-25 [Confirmed]
3. Enforces IP rate limiting via hashed identifier.
   Evidence: apps/web/pages/api/book/event.ts:37-40 [Confirmed]
4. `RegularBookingService` fetches event type and checks reschedule restrictions.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:523-538 [Confirmed]
5. `ensureAvailableUsers` fetches availability and applies pre/post event buffers to busy times.
   Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:103-104 [Confirmed]
6. Runs `checkForConflicts` against buffered busy intervals.
   Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:243-248 [Confirmed]
7. `saveBooking` runs Prisma transaction: inserts `Booking` and `Attendee` records.
   Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:139-147 [Confirmed]

### Simultaneous Requests (Original)
- Two concurrent booking requests targeting identical host slots both pass `ensureAvailableUsers`.
  Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:243-258 [Confirmed]
- Booking creation does not inspect or remove rows in `SelectedSlots`.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:486-550 [Confirmed]
- `saveBooking` performs standard database inserts inside default read-committed Prisma transaction without table locks.
  Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:139-147 [Confirmed]
- Because PostgreSQL lacks a unique constraint across host ID and start/end time, both transactions succeed and commit overlapping bookings.
  Evidence: packages/prisma/schema.prisma:918-930 [Confirmed]
