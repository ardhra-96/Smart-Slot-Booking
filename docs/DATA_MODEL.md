# Smart Slot Booking Data Model
A clean-room relational data specification and database constraint model for the office hours scheduling engine.

```mermaid
erDiagram
    User ||--o{ Schedule : "owns"
    User ||--o{ EventType : "owns"
    User ||--o{ Booking : "hosts"
    User ||--o{ SelectedSlots : "holds"
    Schedule ||--o{ Availability : "defines"
    EventType ||--o{ Booking : "instantiates"
    Booking ||--o{ Attendee : "includes"

    User {
        int id PK
        string email UK
        string name
        string timeZone
    }
    EventType {
        int id PK
        int userId FK
        string title
        string slug
        int length
        int beforeBuffer
        int afterBuffer
    }
    Schedule {
        int id PK
        int userId FK
        string name
        string timeZone
        boolean isDefault
    }
    Availability {
        int id PK
        int scheduleId FK
        int_array days
        time startTime
        time endTime
        date date
    }
    Booking {
        int id PK
        string uid UK
        int eventTypeId FK
        int userId FK
        timestamp startTime
        timestamp endTime
        int beforeBuffer
        int afterBuffer
        string status
    }
    Attendee {
        int id PK
        int bookingId FK
        string email
        string name
        string timeZone
    }
    SelectedSlots {
        int id PK
        int eventTypeId FK
        int userId FK
        timestamp slotUtcStartDate
        timestamp slotUtcEndDate
        string uid UK
        timestamp releaseAt
    }
```

## Rebuild Core Entities

### Model: User
- Purpose: Stores faculty and student account profiles with canonical IANA timezones.
- Constraints: PK `id`, unique `email`.

| Field | Type | Required | Key | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key |
| `email` | String | Yes | Unique | User email address |
| `name` | String | Yes | None | Full display name |
| `timeZone` | String | Yes | None | Canonical IANA timezone (`Asia/Kolkata`) |

### Model: EventType
- Purpose: Configures meeting templates with duration, buffer windows, and slugs.
- Constraints: PK `id`, unique `[userId, slug]`.

| Field | Type | Required | Key | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key |
| `userId` | Int | Yes | FK | Host user reference ID |
| `title` | String | Yes | None | Public event type name |
| `slug` | String | Yes | Unique | URL slug under host handle |
| `length` | Int | Yes | None | Meeting duration in minutes |
| `beforeBuffer` | Int | Yes | None | Buffer minutes preceding meeting |
| `afterBuffer` | Int | Yes | None | Buffer minutes succeeding meeting |

### Model: Schedule
- Purpose: Groups recurring availability and date overrides under an organizer profile.
- Constraints: PK `id`, index `[userId]`.

| Field | Type | Required | Key | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key |
| `userId` | Int | Yes | FK | Schedule owner reference ID |
| `name` | String | Yes | None | User schedule label |
| `timeZone` | String | Yes | None | Host schedule IANA timezone |
| `isDefault` | Boolean | Yes | None | Flag marking default active schedule |

### Model: Availability
- Purpose: Stores weekly recurring availability windows or specific date overrides.
- Constraints: PK `id`, index `[scheduleId]`.

| Field | Type | Required | Key | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key |
| `scheduleId` | Int | Yes | FK | Parent schedule ID |
| `days` | Int[] | Yes | None | Active weekday numbers (0=Sun to 6=Sat) |
| `startTime` | Time | Yes | None | Window opening wall-clock time |
| `endTime` | Time | Yes | None | Window closing wall-clock time |
| `date` | Date | No | None | Specific date override (`YYYY-MM-DD`) |

### Model: Booking
- Purpose: Records confirmed appointments, blocked ranges, and lifecycle statuses.
- Constraints: PK `id`, unique `uid`, PostgreSQL GiST exclusion constraint on host and blocked range.

| Field | Type | Required | Key | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key |
| `uid` | String | Yes | Unique | Public alphanumeric booking UID |
| `eventTypeId` | Int | Yes | FK | Associated event type ID |
| `userId` | Int | Yes | FK | Host user reference ID |
| `startTime` | DateTime | Yes | Index | Meeting start instant in UTC |
| `endTime` | DateTime | Yes | Index | Meeting end instant in UTC |
| `beforeBuffer` | Int | Yes | None | Buffer minutes preceding booking |
| `afterBuffer` | Int | Yes | None | Buffer minutes succeeding booking |
| `status` | BookingStatus | Yes | Index | Enum: CONFIRMED, CANCELLED |

### Model: Attendee
- Purpose: Records student attendee contact information for confirmed meetings.
- Constraints: PK `id`, index `[bookingId]`.

| Field | Type | Required | Key | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key |
| `bookingId` | Int | Yes | FK | Parent booking ID |
| `email` | String | Yes | Index | Student email address |
| `name` | String | Yes | None | Student full name |
| `timeZone` | String | Yes | None | Student IANA timezone |

### Model: SelectedSlots
- Purpose: Records short-lived reservation tokens holding slots during checkout.
- Constraints: PK `id`, unique `uid`.

| Field | Type | Required | Key | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key |
| `eventTypeId` | Int | Yes | None | Event type ID |
| `userId` | Int | Yes | None | Host user ID |
| `slotUtcStartDate` | DateTime | Yes | Index | Reserved slot start instant in UTC |
| `slotUtcEndDate` | DateTime | Yes | Index | Reserved slot end instant in UTC |
| `uid` | String | Yes | Unique | Reservation token UUID |
| `releaseAt` | DateTime | Yes | None | Expiration timestamp in UTC |

## Concurrency Protection & Double Booking Prevention
- Rebuild Database-Enforced Mechanism:
  - Double booking is prevented by a PostgreSQL exclusion constraint utilizing `btree_gist`.
  - Constraint definition:
    ```sql
    ALTER TABLE "Booking" ADD CONSTRAINT "no_overlapping_bookings"
    EXCLUDE USING gist (
      "userId" WITH =,
      tstzrange(
        "startTime" - ("beforeBuffer" || ' minutes')::interval,
        "endTime" + ("afterBuffer" || ' minutes')::interval,
        '[)'
      ) WITH &&
    ) WHERE ("status" != 'CANCELLED');
    ```
  - Mechanism Details:
    - The constraint indexes host ID equality and half-open blocked range overlap.
    - Blocked range equals `[startTime - beforeBuffer, endTime + afterBuffer)`.
    - Any concurrent transaction attempting an overlapping insert triggers an immediate exclusion violation.
    - Transaction rolls back atomically and returns HTTP 409 Conflict with code `SLOT_ALREADY_BOOKED`.

## Reference: Original (Cal.com)
A verified schema specification of the Cal.com scheduling models, database relationships, and table constraints.

```mermaid
erDiagram
    User ||--o{ Schedule : "owns"
    User ||--o{ Booking : "hosts"
    User ||--o{ Availability : "has"
    User ||--o{ EventType : "owns"
    EventType ||--o{ Booking : "instantiates"
    EventType ||--o{ Availability : "defines"
    Schedule ||--o{ Availability : "contains"
    Booking ||--o{ Attendee : "includes"
    Booking ||--o{ BookingSeat : "allocates"
    User ||--o{ SelectedSlots : "holds"
    EventType ||--o{ SelectedSlots : "reserves"

    User {
        int id PK
        string email UK
        string username UK
        string timeZone
    }
    EventType {
        int id PK
        string slug
        int length
        int beforeEventBuffer
        int afterEventBuffer
        int seatsPerTimeSlot
    }
    Schedule {
        int id PK
        int userId FK
        string timeZone
    }
    Availability {
        int id PK
        int scheduleId FK
        int_array days
        time startTime
        time endTime
        date date
    }
    Booking {
        int id PK
        string uid UK
        string idempotencyKey UK
        int userId FK
        timestamp startTime
        timestamp endTime
        string status
    }
    Attendee {
        int id PK
        string email
        string name
        int bookingId FK
    }
    SelectedSlots {
        int id PK
        int userId
        timestamp slotUtcStartDate
        timestamp slotUtcEndDate
        string uid
    }
```

### Scheduling & Booking Core Models (Original)

#### Model: Booking (Original)
- Purpose: Stores scheduled meeting records, time windows, and lifecycle statuses.
  Evidence: packages/prisma/schema.prisma:851-930 [Confirmed]
- Relationships: Belongs to `User` (host), `EventType`; has many `Attendee`, `BookingReference`.
  Evidence: packages/prisma/schema.prisma:856,862,871 [Confirmed]
- Constraints: PK `id`, unique `uid`, `idempotencyKey`, `oneTimePassword`.
  Evidence: packages/prisma/schema.prisma:852,853,855,903 [Confirmed]
- Indexes: `[eventTypeId]`, `[userId]`, `[status]`, `[startTime, endTime, status]`, `[userId, status, startTime]`.
  Evidence: packages/prisma/schema.prisma:918-929 [Confirmed]

| Field | Type | Required | Key | Notes | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary surrogate key | packages/prisma/schema.prisma:852 |
| `uid` | String | Yes | Unique | Public string identifier | packages/prisma/schema.prisma:853 |
| `idempotencyKey` | String | No | Unique | Null in regular bookings | packages/prisma/schema.prisma:855 |
| `userId` | Int | No | FK | Host user reference ID | packages/prisma/schema.prisma:857 |
| `eventTypeId` | Int | No | FK | Event type template ID | packages/prisma/schema.prisma:863 |
| `title` | String | Yes | None | Meeting event title | packages/prisma/schema.prisma:864 |
| `startTime` | DateTime | Yes | Index | Meeting start timestamp in UTC | packages/prisma/schema.prisma:869 |
| `endTime` | DateTime | Yes | Index | Meeting end timestamp in UTC | packages/prisma/schema.prisma:870 |
| `status` | BookingStatus | Yes | Index | Enum: ACCEPTED, CANCELLED, REJECTED, PENDING | packages/prisma/schema.prisma:875 |
| `fromReschedule` | String | No | Index | Previous booking UID when rescheduled | packages/prisma/schema.prisma:888 |

#### Model: Attendee (Original)
- Purpose: Represents invitees and participants linked to a booking.
  Evidence: packages/prisma/schema.prisma:826-841 [Confirmed]
- Relationships: Belongs to `Booking` with cascade on delete.
  Evidence: packages/prisma/schema.prisma:833 [Confirmed]
- Constraints & Indexes: PK `id`; indexes `[email]`, `[bookingId]`, `[email, bookingId]`.
  Evidence: packages/prisma/schema.prisma:827,838-840 [Confirmed]

| Field | Type | Required | Key | Notes | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key | packages/prisma/schema.prisma:827 |
| `email` | String | Yes | Index | Invitee email address | packages/prisma/schema.prisma:828 |
| `name` | String | Yes | None | Invitee full name | packages/prisma/schema.prisma:829 |
| `timeZone` | String | Yes | None | Invitee local IANA timezone | packages/prisma/schema.prisma:830 |
| `bookingId` | Int | No | FK | Reference to parent booking | packages/prisma/schema.prisma:834 |

#### Model: Schedule (Original)
- Purpose: Groups recurring availability and date overrides under an organizer profile.
  Evidence: packages/prisma/schema.prisma:945-958 [Confirmed]
- Relationships: Belongs to `User`, has many `Availability` entries and `EventType` templates.
  Evidence: packages/prisma/schema.prisma:947,949,954 [Confirmed]
- Constraints & Indexes: PK `id`; index `[userId]`.
  Evidence: packages/prisma/schema.prisma:946,957 [Confirmed]

| Field | Type | Required | Key | Notes | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key | packages/prisma/schema.prisma:946 |
| `userId` | Int | Yes | FK | Schedule owner user ID | packages/prisma/schema.prisma:948 |
| `name` | String | Yes | None | User-assigned schedule title | packages/prisma/schema.prisma:952 |
| `timeZone` | String | No | None | Schedule-level IANA timezone | packages/prisma/schema.prisma:953 |

#### Model: Availability (Original)
- Purpose: Stores recurring weekly open hours or specific calendar date overrides.
  Evidence: packages/prisma/schema.prisma:960-976 [Confirmed]
- Relationships: Belongs to `Schedule`, belongs to `User`, belongs to `EventType`.
  Evidence: packages/prisma/schema.prisma:962,964,970 [Confirmed]
- Constraints & Indexes: PK `id`; indexes `[userId]`, `[eventTypeId]`, `[scheduleId]`.
  Evidence: packages/prisma/schema.prisma:961,973-975 [Confirmed]

| Field | Type | Required | Key | Notes | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key | packages/prisma/schema.prisma:961 |
| `userId` | Int | No | FK | Optional direct user link | packages/prisma/schema.prisma:963 |
| `scheduleId` | Int | No | FK | Parent schedule ID | packages/prisma/schema.prisma:971 |
| `days` | Int[] | Yes | None | Weekday numbers (0=Sunday to 6=Saturday) | packages/prisma/schema.prisma:966 |
| `startTime` | DateTime | Yes | None | Opening time (DB Time) | packages/prisma/schema.prisma:967 |
| `endTime` | DateTime | Yes | None | Closing time (DB Time) | packages/prisma/schema.prisma:968 |
| `date` | DateTime | No | None | Date for specific overrides | packages/prisma/schema.prisma:969 |

#### Model: EventType (Original)
- Purpose: Configuration template defining meeting durations, buffers, and scheduling logic.
  Evidence: packages/prisma/schema.prisma:156-306 [Confirmed]
- Relationships: Owned by `User`, belongs to `Team`, links to `Schedule`, has many `Booking` entries.
  Evidence: packages/prisma/schema.prisma:174,180,183 [Confirmed]
- Constraints & Indexes: PK `id`; unique `[userId, slug]`, `[teamId, slug]`; indexes `[userId]`, `[scheduleId]`.
  Evidence: packages/prisma/schema.prisma:295-301 [Confirmed]

| Field | Type | Required | Key | Notes | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key | packages/prisma/schema.prisma:157 |
| `title` | String | Yes | None | Event name shown to bookers | packages/prisma/schema.prisma:159 |
| `slug` | String | Yes | Unique | URL slug under user/team path | packages/prisma/schema.prisma:161 |
| `length` | Int | Yes | None | Event duration in minutes | packages/prisma/schema.prisma:168 |
| `beforeEventBuffer` | Int | Yes | None | Buffer minutes preceding meeting | packages/prisma/schema.prisma:220 |
| `afterEventBuffer` | Int | Yes | None | Buffer minutes succeeding meeting | packages/prisma/schema.prisma:221 |
| `seatsPerTimeSlot` | Int | No | None | Capacity limit for seated events | packages/prisma/schema.prisma:222 |

#### Model: SelectedSlots (Original)
- Purpose: Records temporary slot reservation holds placed by prospective bookers.
  Evidence: packages/prisma/schema.prisma:1437-1448 [Confirmed]
- Relationships: Integer references to `userId` and `eventTypeId` without foreign keys.
  Evidence: packages/prisma/schema.prisma:1439-1440 [Confirmed]
- Constraints: PK `id`; unique `[userId, slotUtcStartDate, slotUtcEndDate, uid]` named `selectedSlotUnique`.
  Evidence: packages/prisma/schema.prisma:1447 [Confirmed]

| Field | Type | Required | Key | Notes | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key | packages/prisma/schema.prisma:1438 |
| `eventTypeId` | Int | Yes | None | Event type identifier | packages/prisma/schema.prisma:1439 |
| `userId` | Int | Yes | None | Host user identifier | packages/prisma/schema.prisma:1440 |
| `slotUtcStartDate` | DateTime | Yes | Unique | Reserved slot start time in UTC | packages/prisma/schema.prisma:1441 |
| `slotUtcEndDate` | DateTime | Yes | Unique | Reserved slot end time in UTC | packages/prisma/schema.prisma:1442 |
| `uid` | String | Yes | Unique | Booker client reservation UUID | packages/prisma/schema.prisma:1443 |
| `releaseAt` | DateTime | Yes | None | Expiration timestamp | packages/prisma/schema.prisma:1444 |

#### Model: User (Original)
- Purpose: Represents registered host account with profile settings and default schedule.
  Evidence: packages/prisma/schema.prisma:401-522 [Confirmed]
- Relationships: Has many `Schedule`, `EventType`, `Booking`, and `Credential` records.
  Evidence: packages/prisma/schema.prisma:461,466,470 [Confirmed]
- Constraints & Indexes: PK `id`; unique `email`, `username`; indexes `[email]`, `[username]`.
  Evidence: packages/prisma/schema.prisma:404,407,514-515 [Confirmed]

| Field | Type | Required | Key | Notes | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | Int | Yes | PK | Primary key | packages/prisma/schema.prisma:402 |
| `email` | String | Yes | Unique | Host primary email address | packages/prisma/schema.prisma:404 |
| `username` | String | No | Unique | Public handle for URLs | packages/prisma/schema.prisma:407 |
| `timeZone` | String | Yes | None | Default IANA user timezone | packages/prisma/schema.prisma:424 |
| `defaultScheduleId`| Int | No | None | Pointer to default schedule | packages/prisma/schema.prisma:462 |

### Constraints Relevant to Double Booking (Original)
- Existing Constraints:
  - `Booking.uid` unique constraint prevents duplicate UIDs.
    Evidence: packages/prisma/schema.prisma:853 [Confirmed]
  - `SelectedSlots` unique constraint on `[userId, slotUtcStartDate, slotUtcEndDate, uid]`.
    Evidence: packages/prisma/schema.prisma:1447 [Confirmed]
- Missing Constraints in Original:
  - No database unique or exclusion constraint on `Booking` for `[userId, startTime, endTime]`.
    Evidence: packages/prisma/schema.prisma:918-930 [Confirmed]
  - No exclusion constraint exists preventing overlapping meeting ranges `[startTime, endTime)`.
    Evidence: packages/prisma/schema.prisma:918-930 [Confirmed]
