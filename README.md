# Smart Slot Booking

## What this is
Clean-room scheduling system reverse engineered from Cal.com.
Provides automated slot calculation, timezone normalization, and conflict-free booking.

## Prerequisites
- Node.js 18+
- Yarn Berry
- Docker and Docker Compose
- PostgreSQL 15+ with btree_gist
- Redis 7+

## Run it
Commands to be confirmed once code lands.
1. `yarn install`
2. `cp .env.example .env`
3. `yarn dx`
4. `yarn dev`

## Run the Killer Tests
- KT1: Host in IST and booker in PST both see correct slots.
- KT2: Buffer time between bookings is respected.
- KT3: Two bookings for the same slot, exactly one succeeds.

## Docs
- [OBSERVATIONS.md](docs/OBSERVATIONS.md): Architectural findings and codebase inventory.
- [PRD.md](docs/PRD.md): Product requirements, scope, and acceptance criteria.
- [ARCHITECTURE.md](docs/ARCHITECTURE.md): Rebuild system architecture and component interactions.
- [DATA_MODEL.md](docs/DATA_MODEL.md): Relational schemas and concurrency exclusion constraints.
- [API.md](docs/API.md): API endpoint contracts and create-booking sequence.
- [GAPS.md](docs/GAPS.md): Concurrency limitations and two architectural improvements.
- [AGENT_LOG.md](docs/AGENT_LOG.md): Chronological prompt audit log and clean-room boundaries.
