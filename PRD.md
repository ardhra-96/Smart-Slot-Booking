# Cal.com Product Requirements Document
A reverse-engineered product requirements document describing the verified capabilities of the existing Cal.com product.

## Problem
Coordinating meetings manually causes scheduling friction, timezone errors, and calendar double-bookings across individual hosts and multi-member organizations.
Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:2580-2585 [Confirmed]

## Target Users
- Individual Host: Sets availability schedules, connects external calendars, and shares booking links.
  Evidence: packages/prisma/schema.prisma:401,945 [Confirmed]
- Booker: Public invitee who selects an open time slot, enters attendee details, and confirms meetings.
  Evidence: packages/prisma/schema.prisma:826,871 [Confirmed]
- Team Member: Host participating in team collective or round-robin event scheduling.
  Evidence: packages/prisma/schema.prisma:738-748 [Confirmed]
- Organization Admin: Configures organization-wide permissions, teams, and membership roles.
  Evidence: packages/prisma/schema.prisma:376-377,740-741 [Confirmed]

## Problem Statement
For hosts who struggle to coordinate meetings across calendars and timezones, Cal.com does automated slot calculation and self-serve booking, unlike manual email exchanges [Likely].
Evidence: packages/features/availability/lib/getUserAvailability.ts:361-400 [Confirmed]

## Core User Flows

### 1. Host Sets Availability
1. Host logs into dashboard and navigates to schedule management.
   Evidence: packages/trpc/server/routers/viewer/availability/schedule/_router.tsx:17-45 [Confirmed]
2. Host edits weekly available days and time intervals.
   Evidence: packages/features/schedules/services/ScheduleService.ts:18-25 [Confirmed]
3. Host optionally inputs specific date overrides or marks entire dates unavailable.
   Evidence: packages/features/schedules/services/ScheduleService.ts:26-34 [Confirmed]
4. Client invokes `viewer.availability.schedule.update` mutation.
   Evidence: packages/trpc/server/routers/viewer/availability/schedule/update.handler.ts:15-22 [Confirmed]
5. Server deletes existing availability records and inserts new availability rows.
   Evidence: packages/features/schedules/services/ScheduleService.ts:121-137 [Confirmed]

### 2. Booker Views Slots
1. Booker opens `/[user]/[type]` public link in browser.
   Evidence: apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx:18-34 [Confirmed]
2. Client queries `viewer.slots.getSchedule` with requested date range and local timezone.
   Evidence: packages/trpc/server/routers/viewer/slots/_router.tsx:18-25 [Confirmed]
3. Server loads host schedule, builds working ranges, and subtracts external and internal busy times.
   Evidence: packages/features/availability/lib/getUserAvailability.ts:511-650 [Confirmed]
4. Server slices open windows into discrete slots and filters out slots reserved by other users.
   Evidence: packages/features/schedules/lib/slots.ts:127-147 [Confirmed], packages/trpc/server/routers/viewer/slots/util.ts:1217-1228 [Confirmed]
5. Client renders selectable time slots grouped by calendar day.
   Evidence: packages/features/bookings/Booker/store.ts:1-50 [Confirmed]

### 3. Booker Creates a Booking
1. Booker selects an available time slot.
   Evidence: packages/features/bookings/Booker/store.ts:1-50 [Confirmed]
2. Client issues `viewer.slots.reserveSlot` mutation to mark slot in `SelectedSlots`.
   Evidence: packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:24-33 [Confirmed]
3. Booker completes booking form with name, email, and responses.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:559-572 [Confirmed]
4. Client submits POST to `/api/book/event`.
   Evidence: apps/web/pages/api/book/event.ts:17-58 [Confirmed]
5. Server verifies rate limits, limits, email verification, and host slot availability.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:581-988 [Confirmed]
6. Prisma transaction inserts new `Booking` and associated `Attendee` rows.
   Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:139-147 [Confirmed]
7. Server schedules triggers, writes external calendar events, and sends confirmation emails.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:2588-2651 [Confirmed]

### 4. Reschedule Booking
1. User opens `/reschedule/[uid]` link.
   Evidence: apps/web/app/reschedule/[uid]/page.tsx:1-50 [Confirmed]
2. Server validates reschedule notice constraints.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:529-538 [Confirmed]
3. User selects a replacement time slot and submits request.
   Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:653-662 [Confirmed]
4. Server sets original booking status to CANCELLED and creates new booking pointing to old UID.
   Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:234,258-262 [Confirmed]

### 5. Cancel Booking
1. User triggers cancellation, issuing DELETE to `/api/cancel`.
   Evidence: apps/web/app/api/cancel/route.ts:16-36 [Confirmed]
2. Server checks CSRF token, rate limit, and booking ownership.
   Evidence: apps/web/app/api/cancel/route.ts:33-54 [Confirmed]
3. Server marks booking status as CANCELLED and dispatches cancellation webhooks.
   Evidence: apps/web/app/api/cancel/route.ts:52-56 [Confirmed]

## Features
| Feature | What it does | Evidence | Status |
| :--- | :--- | :--- | :--- |
| Recurring Weekly Availability | Slices host availability into recurring weekly days and hours | packages/features/schedules/lib/date-ranges.ts:48-88 | Implemented |
| Date Overrides & Days Off | Overrides weekly schedules on specific dates or marks days unavailable | packages/features/schedules/lib/date-ranges.ts:312-318 | Implemented |
| Pre/Post Event Buffers | Adds buffer windows before and after bookings to prevent back-to-back fatigue | packages/features/busyTimes/services/getBusyTimes.ts:135-136 | Implemented |
| Timezone Normalization | Converts host schedule and meeting times to booker local timezone | packages/features/schedules/lib/slots.ts:135-138 | Implemented |
| Temporary Slot Hold | Reserves slots temporarily for active bookers using cookies | packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:24-33 | Implemented |
| Team Round-Robin | Distributes bookings across available hosts in team pool | packages/features/bookings/lib/service/RegularBookingService.ts:1001-1022 | Implemented |
| Seated Meetings | Enables multiple attendees to book identical slots up to seat capacity | packages/features/bookings/lib/service/RegularBookingService.ts:828-860 | Implemented |
| Reschedule & Cancel Flow | Updates meeting states, releases slot, and logs reason | apps/web/app/api/cancel/route.ts:52, packages/features/bookings/lib/handleNewBooking/createBooking.ts:234 | Implemented |

## Time Zone Behavior
- Schedules define host timezone in IANA format.
  Evidence: packages/prisma/schema.prisma:949 [Confirmed]
- Slot generation calculates minute offsets in booker local timezone to respect half-hour boundaries.
  Evidence: packages/features/schedules/lib/slots.ts:135-139 [Confirmed]
- Database stores all timestamps in UTC ISO-8601 format.
  Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:188-189 [Confirmed]

## Buffer Behavior
- Event types store `beforeEventBuffer` and `afterEventBuffer` as integer minutes.
  Evidence: packages/prisma/schema.prisma:186-187 [Confirmed]
- Busy time calculator expands existing meeting start backward and meeting end forward.
  Evidence: packages/features/busyTimes/services/getBusyTimes.ts:177-178 [Confirmed]
- Conflict checker marks slots colliding with buffered windows as unavailable.
  Evidence: packages/features/bookings/lib/conflictChecker/checkForConflicts.ts:39-47 [Confirmed]

## Override / Day-Off Behavior
- Host adds specific calendar date availability entries with optional zero-length duration.
  Evidence: packages/features/schedules/services/ScheduleService.ts:130-134 [Confirmed]
- Overrides overwrite default weekly hours on that date entirely.
  Evidence: packages/features/schedules/lib/date-ranges.ts:312-314 [Confirmed]
- Zero-length date overrides remove all availability for that day, creating days off.
  Evidence: packages/features/schedules/lib/date-ranges.ts:316-318 [Confirmed]

## Double-Booking Behavior
- System checks availability and busy times before booking creation.
  Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:243-248 [Confirmed]
- System uses `SelectedSlots` in slot querying to hide reserved slots.
  Evidence: packages/trpc/server/routers/viewer/slots/util.ts:1217-1228 [Confirmed]
- System lacks database-level mutual exclusion or serializable transaction locks during booking creation.
  Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:139-147 [Confirmed]
- Concurrent booking creation requests for identical slots both succeed, producing double bookings.
  Evidence: packages/prisma/schema.prisma:918-930 [Confirmed]

## Observed Limits
- No unique constraint exists on `Booking` for `[userId, startTime, endTime]`.
  Evidence: packages/prisma/schema.prisma:918-930 [Confirmed]
- Slot reservation logic does not check host group assignees in team events.
  Evidence: packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:80-82 [Confirmed]
- Booking creation route ignores active `SelectedSlots` reservations.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:486-550 [Confirmed]
- Cross-origin iframe embeds cannot read slot reservation cookies if third-party cookies are blocked.
  Evidence: packages/features/bookings/Booker/useSlotReservationId.ts:1-5 [Confirmed]
