# Cal.com API Specification
A verified technical specification of Cal.com core scheduling routes, procedure contracts, and execution workflows.

## Summary of Core Endpoints
| Method | Path or Procedure | Purpose | Auth | Evidence |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/book/event` | Creates an individual booking | Public | apps/web/pages/api/book/event.ts:17-63 [Confirmed] |
| `POST` | `/api/book/recurring-event` | Creates recurring booking instances | Public | apps/web/pages/api/book/recurring-event.ts:29-66 [Confirmed] |
| `query` | `viewer.slots.getSchedule` | Computes available time slots | Public | packages/trpc/server/routers/viewer/slots/_router.tsx:18-25 [Confirmed] |
| `mutation` | `viewer.slots.reserveSlot` | Holds a slot temporarily | Public | packages/trpc/server/routers/viewer/slots/_router.tsx:26-33 [Confirmed] |
| `mutation` | `viewer.slots.removeSelectedSlotMark` | Releases a held slot | Public | packages/trpc/server/routers/viewer/slots/_router.tsx:46-55 [Confirmed] |
| `mutation` | `viewer.availability.schedule.update` | Updates weekly hours and overrides | Authed | packages/trpc/server/routers/viewer/availability/schedule/update.handler.ts:15-22 [Confirmed] |
| `DELETE` | `/api/cancel` | Cancels a confirmed booking | Public / Session | apps/web/app/api/cancel/route.ts:16-70 [Confirmed] |

## Endpoint Specifications

### 1. POST /api/book/event
- Method: `POST`
- Path: `/api/book/event`
- Purpose: Validates, books, and confirms an individual meeting slot.
- Auth: Public (accepts optional authenticated session).
- Who may call: Any public booker or authenticated user.
- Input: JSON body containing `eventTypeId`, `start`, `end`, `name`, `email`, `timeZone`, and `responses`.
  Evidence: packages/features/bookings/lib/dto/types.d.ts:22 [Confirmed]
- Output: Serialized `Booking` record with ID, UID, status, and attendee details.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:2599-2608 [Confirmed]
- Validation: Cloudflare Turnstile token, IP rate limiting, Zod schema parsing, and duration bounds.
  Evidence: apps/web/pages/api/book/event.ts:20-40 [Confirmed]
- Errors: 400 Bad Request, 403 Forbidden (blocked email), 404 Not Found, 429 Too Many Requests.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:589,615,641 [Confirmed]
- Conflict Behavior: Re-evaluates host availability; throws `ErrorCode.NoAvailableUsersFound` if occupied.
  Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:258 [Confirmed]

### 2. POST /api/book/recurring-event
- Method: `POST`
- Path: `/api/book/recurring-event`
- Purpose: Schedules multiple recurring meeting slots in sequence.
- Auth: Public (accepts optional authenticated session).
- Who may call: Any public booker.
- Input: Array of booking objects containing `start`, `end`, and recurring event metadata.
  Evidence: apps/web/pages/api/book/recurring-event.ts:48-59 [Confirmed]
- Output: Array of created `BookingResponse` records.
  Evidence: apps/web/pages/api/book/recurring-event.ts:47 [Confirmed]
- Validation: Turnstile token, IP rate limiting, and recurring count limits.
  Evidence: apps/web/pages/api/book/recurring-event.ts:32-42 [Confirmed]
- Errors: 400 Bad Request, 429 Too Many Requests.
  Evidence: apps/web/pages/api/book/recurring-event.ts:39 [Confirmed]
- Conflict Behavior: Validates initial slot availability before persisting recurring series.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:877-930 [Confirmed]

### 3. query viewer.slots.getSchedule
- Procedure: `viewer.slots.getSchedule`
- Purpose: Returns available booking slots for a requested date range.
- Auth: Public procedure.
- Who may call: Public bookers and embedded widgets.
- Input: `{ startTime: string, endTime: string, eventTypeId?: number, eventTypeSlug?: string, timeZone?: string }`.
  Evidence: packages/trpc/server/routers/viewer/slots/types.ts:7-41 [Confirmed]
- Output: Object mapping `YYYY-MM-DD` date keys to `{ time: Dayjs }` slot arrays.
  Evidence: packages/trpc/server/routers/viewer/slots/util.ts:1246-1250 [Confirmed]
- Validation: Zod schema ensuring valid date strings and `endTime > startTime`.
  Evidence: packages/trpc/server/routers/viewer/slots/types.ts:58-61 [Confirmed]
- Errors: TRPCError with code `BAD_REQUEST` on invalid input.
  Evidence: packages/trpc/server/routers/viewer/slots/types.ts:54-61 [Confirmed]
- Conflict Behavior: Filters out slots conflicting with buffered busy times or active `SelectedSlots`.
  Evidence: packages/trpc/server/routers/viewer/slots/util.ts:1217-1228 [Confirmed]

### 4. mutation viewer.slots.reserveSlot
- Procedure: `viewer.slots.reserveSlot`
- Purpose: Places a temporary hold on a time slot.
- Auth: Public procedure.
- Who may call: Prospective bookers selecting slots.
- Input: `{ eventTypeId: number, slotUtcStartDate: string, slotUtcEndDate: string, _isDryRun?: boolean }`.
  Evidence: packages/trpc/server/routers/viewer/slots/types.ts:63-75 [Confirmed]
- Output: `{ uid: string }` reservation token written to browser cookie.
  Evidence: packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:123-125 [Confirmed]
- Validation: Verifies event type existence and seat capacity.
  Evidence: packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:30-40 [Confirmed]
- Errors: TRPCError with code `NOT_FOUND` if event type does not exist.
  Evidence: packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:36-39 [Confirmed]
- Conflict Behavior: Skips reservation if another user already holds an unexpired reservation.
  Evidence: packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:68-77 [Confirmed]

### 5. mutation viewer.slots.removeSelectedSlotMark
- Procedure: `viewer.slots.removeSelectedSlotMark`
- Purpose: Releases previously reserved slots for a booker.
- Auth: Public procedure.
- Who may call: Public bookers navigating away or deselecting a slot.
- Input: `{ uid: string | null }`.
  Evidence: packages/trpc/server/routers/viewer/slots/types.ts:77-79 [Confirmed]
- Output: `void`.
  Evidence: packages/trpc/server/routers/viewer/slots/_router.tsx:54 [Confirmed]
- Validation: Reads UID from input or cookie.
  Evidence: packages/trpc/server/routers/viewer/slots/_router.tsx:50 [Confirmed]
- Errors: None (no-op if UID is missing).
  Evidence: packages/trpc/server/routers/viewer/slots/_router.tsx:51-53 [Confirmed]
- Conflict Behavior: Deletes matching records from `SelectedSlots`.
  Evidence: packages/trpc/server/routers/viewer/slots/_router.tsx:52 [Confirmed]

### 6. mutation viewer.availability.schedule.update
- Procedure: `viewer.availability.schedule.update`
- Purpose: Updates weekly available windows and date overrides.
- Auth: Authenticated session (`authedProcedure`).
- Who may call: Schedule owner or team administrator.
- Input: `{ scheduleId: number, name?: string, timeZone?: string, isDefault?: boolean, schedule?: Date[][], dateOverrides?: DateRange[] }`.
  Evidence: packages/features/schedules/services/ScheduleService.ts:11-34 [Confirmed]
- Output: Updated schedule record with parsed availability and default indicator.
  Evidence: packages/features/schedules/services/ScheduleService.ts:156-165 [Confirmed]
- Validation: Name non-empty, valid IANA time zone.
  Evidence: packages/features/schedules/services/ScheduleService.ts:13-14 [Confirmed]
- Errors: 401 Unauthorized if caller does not own schedule.
  Evidence: packages/features/schedules/services/ScheduleService.ts:72-76 [Confirmed]
- Conflict Behavior: Deletes existing `Availability` rows for schedule and inserts new rows.
  Evidence: packages/features/schedules/services/ScheduleService.ts:121-137 [Confirmed]

### 7. DELETE /api/cancel
- Method: `DELETE`
- Path: `/api/cancel`
- Purpose: Cancels a scheduled booking.
- Auth: Public with CSRF token (or session user).
- Who may call: Meeting booker or host holding booking UID.
- Input: JSON body `{ uid: string, cancellationReason?: string, csrfToken?: string }`.
  Evidence: apps/web/app/api/cancel/route.ts:23-26 [Confirmed]
- Output: `{ success: boolean }`.
  Evidence: apps/web/app/api/cancel/route.ts:67 [Confirmed]
- Validation: UID format check, CSRF token verification, and rate limiting.
  Evidence: apps/web/app/api/cancel/route.ts:26-47 [Confirmed]
- Errors: 400 Bad Request on invalid JSON or bad CSRF, 429 on rate limit.
  Evidence: apps/web/app/api/cancel/route.ts:21,35,44 [Confirmed]
- Conflict Behavior: Sets booking status to `CANCELLED`, opening future slot availability.
  Evidence: apps/web/app/api/cancel/route.ts:52-56 [Confirmed]

## Booking Creation Sequence
1. Handler receives POST request at `/api/book/event` and extracts client IP.
   Evidence: apps/web/pages/api/book/event.ts:18 [Confirmed]
2. Validates Cloudflare Turnstile token if enabled.
   Evidence: apps/web/pages/api/book/event.ts:20-25 [Confirmed]
3. Executes bot detection evaluation against request headers.
   Evidence: apps/web/pages/api/book/event.ts:32-35 [Confirmed]
4. Enforces IP rate limiting via hashed identifier.
   Evidence: apps/web/pages/api/book/event.ts:37-40 [Confirmed]
5. Retrieves optional user session.
   Evidence: apps/web/pages/api/book/event.ts:42 [Confirmed]
6. `RegularBookingService` fetches event type and checks reschedule restrictions if rescheduling.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:523-538 [Confirmed]
7. Parses booking request body against event type schema.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:545-549 [Confirmed]
8. Checks if booker email is blocked and validates active booking limits.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:581-610 [Confirmed]
9. Validates booker email verification code if required by event.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:612-629 [Confirmed]
10. Validates booking start time bounds and event duration.
    Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:764-778 [Confirmed]
11. Loads hosts and filters round-robin and fixed attendees.
    Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:791-806 [Confirmed]
12. Checks seat capacity if event type defines `seatsPerTimeSlot`.
    Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:828-860 [Confirmed]
13. `ensureAvailableUsers` fetches availability and applies pre/post event buffers to busy times.
    Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:103-104 [Confirmed]
14. Runs `checkForConflicts` against buffered busy intervals.
    Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:243-248 [Confirmed]
15. Throws `ErrorCode.NoAvailableUsersFound` if all hosts are unavailable.
    Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:258 [Confirmed]
16. Selects assigned host via `luckyUserService` for round-robin meetings.
    Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:1034-1044 [Confirmed]
17. `saveBooking` runs Prisma transaction: updates old booking if rescheduling and inserts new `Booking` and `Attendee` records.
    Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:139-147 [Confirmed]
18. Dispatches calendar invitations, triggers webhooks, and queues notification tasks.
    Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:2588-2651 [Confirmed]

## Simultaneous Requests
- Two concurrent booking requests targeting the identical host slot both pass `ensureAvailableUsers` because neither booking is yet committed to the database.
  Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:243-258 [Confirmed]
- Booking creation does not inspect or remove rows in `SelectedSlots`.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:486-550 [Confirmed]
- `saveBooking` performs standard database inserts inside a default read-committed Prisma transaction without table locks.
  Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:139-147 [Confirmed]
- Because PostgreSQL lacks a unique constraint across host ID and start/end time, both transactions succeed and commit overlapping bookings.
  Evidence: packages/prisma/schema.prisma:918-930 [Confirmed]
