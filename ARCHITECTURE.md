# CCCB Relay — Architecture

This document describes the design of the CCCB Relay system: what it's built from, how the pieces fit together, and why the key decisions were made.

The source code lives in a private repository.

---

## The Stack

**Next.js + Supabase + Vercel.**

The choice was deliberate: no custom backend server. Supabase's auto-generated REST API handles all data reads and writes, with Row-Level Security (RLS) policies enforcing access control at the database layer. The result is a system where:

- Business logic lives in the database, not in application code
- Adding a new access rule means writing a SQL policy, not modifying an API endpoint
- Deployment is a push to `main` — Vercel handles the rest

**Polling, not WebSockets.** The app refreshes ticket lists every 30 seconds and the detail panel every 15 seconds. At the scale this system operates — a single office, ~50 tickets/day — polling is simpler, cheaper, and easier to reason about than a persistent connection. The switch to Supabase Realtime would require no schema changes if volume ever grows to warrant it.

---

## The Two Core Entities

### Client
A family record. Name, zip code, preferred language, a state-assigned family ID (FID), and a default assigned specialist. Client records are persistent and editable by any authenticated user — there's no ownership. A client record belongs to the team, not to an individual.

### Ticket
The unit of work. One ticket per inbound contact. Every ticket belongs to a client, carries a category (what it's about), an urgency flag (manual boolean), a status (open → claimed → resolved), and a full history of who touched it and when.

These are two separate entities by design. The split happened when it became clear that a client might have dozens of contacts over years — and that the history of those contacts was as important as the client record itself.

**Resolved is terminal.** Tickets can't be reopened. "Reopen as New" creates a new linked ticket with the same client data and a breadcrumb back to the original. This preserves audit trail integrity: a resolved ticket is a closed chapter; a follow-up is a new one that references the old.

---

## The Data Model (Conceptual)

Nine tables, organized into layers:

**Core entities**
- `clients` — one record per family
- `tickets` — one record per inbound contact

**History and collaboration**
- `ticket_events` — append-only audit trail (created, claimed, unclaimed, resolved, reassigned). Every state change is recorded with actor and timestamp.
- `ticket_comments` — timestamped, attributed notes per ticket. Append-only. Separate from events because notes are human context; events are mechanical state changes.
- `ticket_reads` — tracks when each user last viewed a ticket; drives unread indicators.

**Configuration**
- `users` — app profiles, roles, active/inactive status
- `territories` — admin-managed zip code groupings
- `categories` — admin-managed dropdown values for ticket type

**Communication**
- `bulletin_posts` — staff-wide short announcements, separate from ticket work

The separation between events and comments is intentional. Events are system-generated facts ("claimed at 10:32 AM by Rose"). Comments are human-written context ("client mentioned the voucher expires on the 15th"). They have different lifespans, display patterns, and potential edit/delete rules. Conflating them would mean either losing the structured audit trail or burdening the human note-taking flow with system constraints.

---

## Routing

When a receptionist creates a ticket, the system auto-assigns a specialist based on a sequence of lookups:

1. **Zip code** → which territory does that zip belong to?
2. **Territory** → which specialists cover that territory?
3. **Name range** → if a territory has specialists split across alphabetical ranges (e.g., A–K / L–Z), match the client's last name to the right one
4. **Department awareness** → prefer to auto-assign within the same department (Family Service vs. I&R) as the person creating the ticket

The result is shown to the receptionist as a hint ("Auto-assigned: Rose · South Boston") with a one-click override. The receptionist always has the final word. If no match is found, the ticket lands in the general pool and a supervisor routes it manually.

The guiding principle: **routing is a suggestion, not a constraint.** The system does the lookup; the human confirms or overrides. This respects the reality that not all routing is mechanical — sometimes a family has a relationship with a specific specialist that a new receptionist wouldn't know about.

---

## The Role Model

Five roles: `receptionist`, `specialist`, `i_and_r`, `supervisor`, `admin`.

The guiding philosophy was **visibility without surveillance.** All roles can see all tickets. Role determines where you land when you log in and what actions you can take — not what information you can see.

The reasoning: if a receptionist is trying to help a family and wants to check the status of a ticket, they shouldn't hit an access wall. Everyone on the team is working toward the same goal. Gatekeeping visibility doesn't serve that.

Elevated capabilities:
- **Supervisor** — can reassign, unclaim, or resolve any ticket. Can see analytical views of team workload.
- **Admin** — everything a supervisor can do, plus system configuration (territories, categories, users) and the metrics dashboard.

---

## The Ticket Lifecycle

```
OPEN → CLAIMED → RESOLVED
```

**Open:** Created by front desk. Assigned to a specialist or territory pool.

**Claimed:** A specialist picks it up. Claiming is a public act — everyone can see who has it and since when. Stale tickets (claimed but no activity for 7+ days) surface in the Team View so supervisors can intervene.

**Resolved:** Specialist marks it done, optionally with a resolution note. Terminal state.

Resolved tickets are soft-archived after 12 months — hidden from normal views but preserved in the database. Every transition is recorded in the audit trail. There is no way to resolve a ticket without leaving a record.

---

## What Was Left Out of V1 (and Why)

**Email notifications** — deferred. An external transactional email dependency isn't warranted at current scale. In-app unread indicators and 30-second polling cover the awareness gap for an office where staff are actively logged in during work hours.

**WebSockets** — overkill for a single-office deployment. Polling is simpler, cheaper, and easier to debug. The tradeoff is explicit latency (up to 30 seconds), which is acceptable here.

**Client self-intake** — a V2 concept. A public-facing form (no login required) in multiple languages that creates a ticket directly in the system. The front desk remains the quality gate — they review, confirm, and route every submission. The vision: a QR code on a business card, in English, Spanish, and Haitian Creole. Point camera, tap link, fill four fields. No app, no account, no instructions. For families who have spent years navigating systems that weren't built for them, a form in their language that works on their schedule signals something different.

**External system integration** — no state system API, no scraping. Client records build organically through intake.

**Reporting and export** — the admin metrics panel covers basic volume tracking. Full export is V2.

The scope decisions follow a single principle: **ship a complete, reliable V1 before adding features.** A system that does fewer things well is more useful than one that does many things inconsistently.
