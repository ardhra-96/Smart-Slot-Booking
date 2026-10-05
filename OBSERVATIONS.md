# Cal.com Codebase Observations
A verified factual record of the Cal.com codebase architecture, behaviors, and implementations.

## Tech Stack
| Layer | Technology | Evidence |
| :--- | :--- | :--- |
| Language | TypeScript 5.x | package.json:88 [Confirmed] |
| Web Framework | Next.js 16.2.3 (App Router & Pages Router) | apps/web/package.json:110 [Confirmed] |
| Frontend Library | React 18.2.0 | apps/web/package.json:123 [Confirmed] |
| API Layer | tRPC v10 & Next.js API Routes | packages/trpc/server/trpc.ts:4 [Confirmed] |
| Database | PostgreSQL | docker-compose.yml:15-22 [Confirmed] |
| ORM | Prisma Client 6.16.1 | packages/prisma/package.json:29 [Confirmed] |
| Monorepo Tooling | Turborepo & Yarn Berry 3.4.1 | package.json:30 [Confirmed], apps/web/package.json:31 [Confirmed] |
| Date & Time Engine | Day.js with UTC and Timezone plugins | packages/features/schedules/lib/date-ranges.ts:2 [Confirmed] |
| Auth Framework | NextAuth.js 4.24.13 | apps/web/package.json:111 [Confirmed] |
| Cache & Key-Value | Redis via @upstash/redis | apps/web/package.json:83 [Confirmed], docker-compose.yml:28 [Confirmed] |

## Repository Layout
- Monorepo organized via Yarn workspaces.
  Evidence: package.json:5-16 [Confirmed]
- `apps/web`: Next.js web application handling public booking and host settings.
  Evidence: apps/web/package.json:2 [Confirmed]
- `apps/api/v2`: Standalone REST API v2 platform service.
  Evidence: apps/api/v2/package.json:2 [Confirmed]
- `packages/prisma`: Database schema definitions, Prisma migrations, and seeds.
  Evidence: packages/prisma/package.json:2 [Confirmed]
- `packages/features`: Domain services for bookings, availability, schedules, and auth.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:2585 [Confirmed]
- `packages/trpc`: tRPC routers, procedures, and middleware definitions.
  Evidence: packages/trpc/server/routers/_app.ts:1 [Confirmed]
- `packages/lib`: Shared helper utilities, error definitions, and constants.
  Evidence: packages/lib/constants.ts:1 [Confirmed]

## How It Runs
- Run `yarn dx` to start docker containers, run database migrations, and seed data.
  Evidence: packages/prisma/package.json:19 [Confirmed]
- Run `yarn dev` to launch the Next.js development server with Turbopack.
  Evidence: package.json:47 [Confirmed], apps/web/package.json:10 [Confirmed]
- Requires a PostgreSQL database container.
  Evidence: docker-compose.yml:13-25 [Confirmed]
- Requires a Redis service container for caching and rate limiting.
  Evidence: docker-compose.yml:26-36 [Confirmed]

## Environment Variables
- `DATABASE_URL`: Primary PostgreSQL connection string.
  Evidence: .env.example:17 [Confirmed]
- `DATABASE_DIRECT_URL`: Direct PostgreSQL connection string for migrations bypassing poolers.
  Evidence: .env.example:20 [Confirmed]
- `NEXT_PUBLIC_WEBAPP_URL`: Base public URL for the web application.
  Evidence: .env.example:28 [Confirmed]
- `NEXTAUTH_SECRET`: Secret key used to encrypt NextAuth session tokens.
  Evidence: .env.example:59 [Confirmed]
- `NEXTAUTH_URL`: Canonical URL for NextAuth authentication callbacks.
  Evidence: .env.example:56 [Confirmed]
- `CALENDSO_ENCRYPTION_KEY`: Symmetric key for encrypting credentials and integrations.
  Evidence: .env.example:76 [Confirmed]
- `CRON_API_KEY`: API key protecting scheduled internal endpoints.
  Evidence: .env.example:67 [Confirmed]
- `NEXT_PUBLIC_AVAILABILITY_SCHEDULE_INTERVAL`: Integer interval for slot step calculation.
  Evidence: packages/features/schedules/lib/slots.ts:113 [Confirmed]
- `NEXT_PUBLIC_CLOUDFLARE_USE_TURNSTILE_IN_BOOKER`: Toggle flag for Cloudflare Turnstile bot validation.
  Evidence: apps/web/pages/api/book/event.ts:20 [Confirmed]

## Key Files
- `packages/features/availability/lib/getUserAvailability.ts`: Core availability calculation service.
  Evidence: packages/features/availability/lib/getUserAvailability.ts:1 [Confirmed]
- `packages/features/schedules/lib/date-ranges.ts`: Date range builder and override processor.
  Evidence: packages/features/schedules/lib/date-ranges.ts:1 [Confirmed]
- `packages/features/schedules/lib/slots.ts`: Slot slicing and timezone boundary generator.
  Evidence: packages/features/schedules/lib/slots.ts:1 [Confirmed]
- `packages/features/busyTimes/services/getBusyTimes.ts`: Aggregator for calendar and booking busy windows.
  Evidence: packages/features/busyTimes/services/getBusyTimes.ts:26 [Confirmed]
- `packages/features/bookings/lib/service/RegularBookingService.ts`: Booking creation and orchestration service.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:2585 [Confirmed]
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`: Prisma transaction for writing new bookings.
  Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:61 [Confirmed]
- `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts`: Interval collision detection utility.
  Evidence: packages/features/bookings/lib/conflictChecker/checkForConflicts.ts:10 [Confirmed]
- `apps/web/pages/api/book/event.ts`: HTTP route handler for creating individual bookings.
  Evidence: apps/web/pages/api/book/event.ts:17 [Confirmed]
- `packages/trpc/server/routers/viewer/slots/_router.tsx`: tRPC router for schedule and slot discovery.
  Evidence: packages/trpc/server/routers/viewer/slots/_router.tsx:17 [Confirmed]
- `apps/web/app/api/cancel/route.ts`: App router route handler for booking cancellation.
  Evidence: apps/web/app/api/cancel/route.ts:16 [Confirmed]

## Availability Calculation
- Host schedule is located via `detectEventTypeScheduleForUser`.
  Evidence: packages/features/availability/lib/getUserAvailability.ts:401-413 [Confirmed]
- `buildDateRanges` builds base availability intervals from weekly working hours.
  Evidence: packages/features/availability/lib/getUserAvailability.ts:511-518 [Confirmed]
- Internal bookings and external calendar events are collected as busy times.
  Evidence: packages/features/availability/lib/getUserAvailability.ts:586-605 [Confirmed]
- Busy time windows are subtracted from availability intervals using interval subtraction.
  Evidence: packages/features/availability/lib/getUserAvailability.ts:648-649 [Confirmed]
- `buildSlotsWithDateRanges` divides remaining windows into discrete bookable slots.
  Evidence: packages/features/schedules/lib/slots.ts:71-100 [Confirmed]

## Time-Zone Handling
- Schedules store host time zone as an IANA time zone string.
  Evidence: packages/prisma/schema.prisma:949 [Confirmed]
- Slot generation converts UTC start timestamps to booker time zone prior to minute rounding.
  Evidence: packages/features/schedules/lib/slots.ts:135-138 [Confirmed]
- Booking requests submit timestamps in ISO-8601 UTC strings.
  Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:188-189 [Confirmed]
- PostgreSQL stores all booking start and end timestamps in UTC.
  Evidence: packages/prisma/schema.prisma:869-870 [Confirmed]

## Buffer Handling
- Event types store integer minutes for `beforeEventBuffer` and `afterEventBuffer`.
  Evidence: packages/prisma/schema.prisma:186-187 [Confirmed]
- Busy time aggregator calculates combined buffer times before and after existing meetings.
  Evidence: packages/features/busyTimes/services/getBusyTimes.ts:135-136 [Confirmed]
- Existing booking start is shifted earlier and end shifted later by buffer amounts.
  Evidence: packages/features/busyTimes/services/getBusyTimes.ts:177-178 [Confirmed]
- Conflict detection checks slot collision against buffered busy intervals.
  Evidence: packages/features/bookings/lib/conflictChecker/checkForConflicts.ts:39-47 [Confirmed]

## Date Overrides / Days Off
- Overrides are saved as `Availability` records with a populated `date` column.
  Evidence: packages/prisma/schema.prisma:967 [Confirmed]
- `buildDateRanges` replaces recurring working hours on matching dates with override ranges.
  Evidence: packages/features/schedules/lib/date-ranges.ts:312-314 [Confirmed]
- Overrides with matching start and end times designate full unavailable days off.
  Evidence: packages/features/schedules/lib/date-ranges.ts:316-318 [Confirmed]

## Booking Creation
- HTTP handler accepts booking payload and validates bot tokens and rate limits.
  Evidence: apps/web/pages/api/book/event.ts:20-40 [Confirmed]
- `RegularBookingService` verifies booker blocks, active booking limits, and event duration.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:581-610,772-778 [Confirmed]
- `ensureAvailableUsers` checks availability and interval conflicts for assigned hosts.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:936-946 [Confirmed]
- `saveBooking` runs a Prisma transaction creating `Booking` and `Attendee` records.
  Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:139-147 [Confirmed]

## Conflict Prevention
- `checkForConflicts` compares slot start and end timestamps against sorted busy intervals.
  Evidence: packages/features/bookings/lib/conflictChecker/checkForConflicts.ts:39-47 [Confirmed]
- Slot generation filters out slots conflicting with external calendars or existing bookings.
  Evidence: packages/trpc/server/routers/viewer/slots/util.ts:1217-1228 [Confirmed]
- Booking submission re-validates collisions via `checkForConflicts` inside `ensureAvailableUsers`.
  Evidence: packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:243-248 [Confirmed]

## Concurrency Behavior
- `reserveSlot` mutation writes temporary reservations to `SelectedSlots` table.
  Evidence: packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:84-100 [Confirmed]
- Reservations expire after `MINUTES_TO_BOOK` duration.
  Evidence: packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts:29 [Confirmed]
- `RegularBookingService.createBooking` does not check or delete `SelectedSlots` records.
  Evidence: packages/features/bookings/lib/service/RegularBookingService.ts:486-550 [Confirmed]
- Database schema does not enforce a unique constraint on host ID and meeting time slot.
  Evidence: packages/prisma/schema.prisma:918-930 [Confirmed]
- Simultaneous booking requests for the same slot can create overlapping bookings.
  Evidence: packages/features/bookings/lib/handleNewBooking/createBooking.ts:139-147 [Confirmed]

## Authentication and Authorization
- User sessions are managed via NextAuth.js JWT or database sessions.
  Evidence: apps/web/pages/api/auth/[...nextauth].ts:9-16 [Confirmed]
- Authenticated tRPC procedures require session verification through `isAuthed` middleware.
  Evidence: packages/trpc/server/procedures/authedProcedure.ts:27 [Confirmed]
- Slot lookup and slot reservations run through unauthenticated `publicProcedure`.
  Evidence: packages/trpc/server/routers/viewer/slots/_router.tsx:18-34 [Confirmed]

## External Services
- Google Calendar API for two-way calendar sync.
  Evidence: apps/web/package.json:58 [Confirmed]
- Stripe API for payment collection during booking.
  Evidence: apps/web/package.json:79-80 [Confirmed]
- Daily.co for integrated Cal Video conferencing rooms.
  Evidence: apps/web/package.json:50-51 [Confirmed]
- Nodemailer for email delivery.
  Evidence: apps/web/package.json:116 [Confirmed]
- Cloudflare Turnstile for booker bot validation.
  Evidence: apps/web/pages/api/book/event.ts:20-25 [Confirmed]

## UI Behavior
- Public booking page loads under Next.js App Router at `/[user]/[type]`.
  Evidence: apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx:18-34 [Confirmed]
- Client state and calendar grid selection are managed by the `Booker` store.
  Evidence: packages/features/bookings/Booker/store.ts:1-50 [Confirmed]
- Reschedule flow loads existing booking details at `/reschedule/[uid]`.
  Evidence: apps/web/app/reschedule/[uid]/page.tsx:1-50 [Confirmed]
