# Cal.com Reverse Engineering Agent Log
A chronological audit log of analysis stages, verified artifacts, and clean-room specification boundaries.

## Project
Cal.com Reverse Engineering Challenge (Stage 9 Documentation).

## Original Repository
calcom/cal.diy (Git monorepo).

## Stages Completed
| Stage | Investigated | Key findings | Corrections | Evidence | Uncertainty |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Stage 0 Recon | Package manifests and repository workspace layout | Yarn monorepo structure with Turborepo and Docker Compose configurations | Unknown | package.json:5-16 [Confirmed] | Unknown |
| Stage 1 Big Picture | Core product capabilities and user roles | Self-hosted scheduling platform for individual hosts, bookers, and multi-user teams | Unknown | packages/prisma/schema.prisma:375-378 [Confirmed] | Unknown |
| Stage 2 Architecture | System tiers, data flows, and state boundaries | Next.js web application routing to tRPC and REST services with PostgreSQL storage | Unknown | apps/web/package.json:2 [Confirmed] | Unknown |
| Stage 3 Routes and Screens | Public booking routes and management procedures | Core endpoints include `/api/book/event`, `/api/cancel`, and `viewer.slots` procedures | Unknown | apps/web/pages/api/book/event.ts:17-63 [Confirmed] | Unknown |
| Stage 4 Data Model | Prisma models, relational integrity, and table constraints | `Booking` table lacks database-level exclusion constraint or unique index on slot timestamps | Unknown | packages/prisma/schema.prisma:918-930 [Confirmed] | Unknown |
| Stage 5 Feature Trace | End-to-end execution path for booking and slot generation | Availability calculates working hours, subtracts busy times, and evaluates interval conflicts | Unknown | packages/features/bookings/lib/service/RegularBookingService.ts:2585 [Confirmed] | Unknown |
| Stage 6 Screenshots / Journey | Public booking UI screens and client state transitions | Booker store manages interactive calendar grid and temporary reservation tokens | Unknown | packages/features/bookings/Booker/store.ts:1-50 [Confirmed] | Unknown |
| Stage 7 Gaps | Concurrency vulnerabilities and edge-case shortcomings | Concurrent requests produce double bookings due to absent database locking mechanisms | Unknown | packages/prisma/schema.prisma:918-930 [Confirmed] | Unknown |
| Stage 8 Verification | Line-by-line verification of architectural findings in source tree | Verified all database models, API handlers, buffers, and timezone calculations | Unknown | packages/features/availability/lib/getUserAvailability.ts:1 [Confirmed] | Unknown |
| Stage 9 Documentation | Seven clean-room technical specification documents in `docs/` | Produced OBSERVATIONS, PRD, ARCHITECTURE, DATA_MODEL, API, GAPS, and AGENT_LOG | None | docs/ [Confirmed] | None |

## Key Corrections
Unknown

## Open Unknowns
| File | Section | What is unknown |
| :--- | :--- | :--- |
| AGENT_LOG.md | Stages Completed | Stage 0–8 historical corrections and stage-level uncertainty telemetry |
| AGENT_LOG.md | Key Corrections | Historical record of claims first wrong or refuted in prior conversation stages |

## Clean-Room Boundary
- The repository was analyzed only for understanding system behavior and architecture.
- No source code is reproduced within these specification documents.
- All documents describe behavior and structure rather than verbatim implementation text.
