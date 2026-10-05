# Cal.com System Gaps
A verified register of architectural weaknesses, concurrency limitations, and edge-case deficiencies in Cal.com.

| # | Type | What is wrong/missing | Evidence | Who it hurts | Suggested fix | Severity |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Concurrency | Missing database exclusion constraint or locks on `Booking` allows concurrent requests to double-book identical host slots | packages/prisma/schema.prisma:918-930 [Confirmed] | Hosts and bookers facing conflicting meetings | Add a PostgreSQL exclusion constraint or unique index on host ID and time range | Critical |
| 2 | Concurrency | `RegularBookingService` does not check or delete temporary reservation holds in `SelectedSlots` when creating confirmed bookings | packages/features/bookings/lib/service/RegularBookingService.ts:486-550 [Confirmed] | Bookers whose held slots get booked by outside API requests | Validate active reservation token and delete matching `SelectedSlots` records on booking creation | High |
| 3 | Data | The `idempotencyKey` column on `Booking` is set to null, leaving client duplicate submission protection non-functional | packages/features/bookings/lib/service/RegularBookingService.ts:1539 [Confirmed] | Bookers creating accidental duplicate bookings from double-clicks | Generate idempotencyKey from request hash or accept client idempotency header | High |
| 4 | Correctness | Team event slot reservation only blocks the event creator, ignoring assignees and routed team members | packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:80-82 [Confirmed] | Team members whose calendars are not reserved during multi-host booking flows | Reserve slots for routed team members rather than only the event creator | High |
| 5 | Correctness | Schedule updates execute a destructive `deleteMany` and `createMany` on `Availability`, risking availability deletion on failure | packages/features/schedules/services/ScheduleService.ts:121-137 [Confirmed] | Hosts whose availability windows are erased during schedule metadata updates | Update schedule properties independently without recreating all availability rows | Medium |
| 6 | Timezone | Date override range calculation uses day padding arithmetic to avoid UTC-local date mismatch errors | packages/features/schedules/lib/date-ranges.ts:277-281 [Confirmed] | Hosts experiencing false unavailable errors near midnight UTC boundaries | Parse override dates directly in the schedule target timezone | Medium |
| 7 | UX | Slot reservation cookie cannot be read inside third-party iframe embeds when cross-site cookies are blocked | packages/features/bookings/Booker/useSlotReservationId.ts:1-5 [Confirmed] | Prospective bookers using embedded booking widgets on third-party domains | Persist reservation identifier in top window via postMessage synchronization | Medium |
| 8 | UX | Hardcoded English locale in translation service causes guest notifications to ignore non-English preferences | packages/features/bookings/lib/service/RegularBookingService.ts:636-637 [Confirmed] | Non-English attendees receiving English guest notification emails | Pass attendee locale to getTranslation instead of hardcoding English | Low |
| 9 | Missing feature | The `Booking.scheduledJobs` field is deprecated but still populated with empty arrays on creation | packages/prisma/schema.prisma:891 [Confirmed], packages/features/bookings/lib/service/RegularBookingService.ts:1552 [Confirmed] | Developers maintaining redundant database columns | Drop deprecated scheduledJobs column and migrate existing records to scheduledTriggers | Low |
| 10 | Documentation drift | Source code comment indicates `slotsRouter.getSchedule` should be named `getAvailableSlots` to match functionality | packages/trpc/server/routers/viewer/slots/_router.tsx:16 [Confirmed] | Developers navigating inconsistent procedure nomenclature | Rename procedure to getAvailableSlots with an alias for backwards compatibility | Low |

## Improvement 1
- Name: Database Exclusion Constraint for Concurrent Overlap Prevention
- Problem: Cal.com lacks database-level mutual exclusion on bookings, allowing concurrent requests to create overlapping meetings for the same host.
- Why it matters: Concurrent student bookings for the same office hours slot cause double bookings and scheduling conflicts.
- Implementation direction: Define a PostgreSQL exclusion constraint using GiST on host ID and meeting time ranges with pre/post buffers included.
- Killer Test / requirement supported: KT3 (Two bookings for the same slot: exactly one succeeds).

## Improvement 2
- Name: Atomic Reservation Token Validation and Hold Release
- Problem: Cal.com stores temporary reservation holds in SelectedSlots but ignores them during booking creation, allowing other requests to take reserved slots.
- Why it matters: A booker completing booking details can lose their selected slot to a racing API call.
- Implementation direction: Require a reservation token on booking creation and atomically validate and delete the matching SelectedSlots row during booking insertion.
- Killer Test / requirement supported: KT3 (Two bookings for the same slot: exactly one succeeds) and basic successful booking flow.
