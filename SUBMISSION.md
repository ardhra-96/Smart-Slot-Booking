# Submission Summary

| Field | Value |
| :--- | :--- |
| Team name | BAKLAVA |
| Team ID | DBG-922 |
| Members | Ajay Srinivasan , Janvi Ramachandran , Ardhra Sagar |
| Card | Smart Slot Booking, Workflow & Trust, King of Hearts, Brutal x1.2 |
| Original repository | https://github.com/calcom/cal.diy |
| Commit studied | 54343aa685ae8f33159d2f485ec4a57bad5c574a |
| Improvement 1 | Database Exclusion Constraint for Concurrent Overlap Prevention: Define a PostgreSQL exclusion constraint using GiST on host ID and meeting time ranges with pre/post buffers included. |
| Improvement 2 | Atomic Reservation Token Validation and Hold Release: Require a reservation token on booking creation and atomically validate and delete the matching SelectedSlots row during booking insertion. |
| Killer Test status | KT1: Not yet run, KT2: Not yet run, KT3: Not yet run |
