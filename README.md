# CCCB Relay

A lightweight internal ticketing system built for [Childcare Choices of Boston](https://childcarechoicesofboston.org) — a Boston nonprofit that connects low-income families with subsidized childcare.

Every inbound client contact — phone call, email, or walk-in — becomes a ticket. Tickets are routed to the right team member, tracked through to resolution, and leave a permanent record.
Internal transfers of information can also become a ticket, enabling persistant tracking and shared context of previously implicit handoffs.

The system is designed to reduce the amount of time and energy spent on reconstructing information that was previously known or collected, by keeping a lightweight, easy to search, and persistent record that all team members can see, change, and comment on.

**Built by:** Warren Zhao, former employee at Childcare Choices of Boston  
**Stack:** Next.js · Supabase · Vercel  
**Status:** Deployed to production

---

## How It Works

A **ticket** is the universal unit of work. Every transfer of information can become a ticket issuance if desired. Every ticket has:
- A **client** it belongs to
- A **category** (what it's about)
- A **status** (open → claimed → resolved)
- An **assigned team member** or territory pool it's waiting in
- A full **audit trail** of every action taken

The system has five roles, built on a simple principle: **everyone sees everything, but not everyone can do everything.**

| Role | Default View | What they do |
|------|-------------|--------------|
| Front Desk | Intake | Search clients, create and route tickets |
| Specialist | My Queue | Claim tickets, add notes, resolve cases |
| I&R | My Queue | Same as Specialist — different territory routing |
| Supervisor | Team View | See all tickets, reassign, manage team workload |
| Admin | Team View | Everything above, plus system configuration and metrics |

---

## Key Features

### Intake
- Order-agnostic name search — "Julie Occeant" finds "Occeant, Julie"
- Auto-assigns the right specialist based on zip code, territory, and alphabetical name ranges
- Form designed for one-handed use during a live phone call

### My Queue
- Personal ticket list + shared territory pool in one view
- "Created Tickets" sidebar — your outbox, tracking what you sent into the system
- Yellow unread border on tickets with activity since your last view
- 30-second background polling

### Team View
- Compact single-line rows — 4× more tickets visible per screen than card layouts
- Instant text search by client name, ticket number, or specialist
- Supervisor tools: group by member, stale ticket detection (7+ days inactive), reassign any ticket

### Ticket Detail
- Opens as a side panel — your queue stays visible behind it
- Full comment thread, full audit trail
- "Reopen as New" creates a linked follow-up instead of reopening a resolved ticket

### Admin
- Configuration panel for territories, categories, and users
- Metrics dashboard: volume by category/channel, time-to-claim, time-to-resolve, per-member breakdown
- Soft-archive: resolved tickets over 12 months old are hidden from views but preserved for audit

### Bulletin Board
- Staff-wide short announcements with 24-hour auto-expiry
- Supervisor posts pin to the top
- Ambient team communication — separate from ticket work

---

## Screenshots

*(Coming soon)*

<!--
To add screenshots, create an `assets/` folder in this repo and drop in your images.

Suggested shots:
  - Intake screen (client search + new ticket form)     → assets/intake.png
  - My Queue (ticket list + created tickets sidebar)    → assets/queue.png
  - Team View (compact rows + group by member)          → assets/team-view.png
  - Ticket Detail (side panel + comment thread)         → assets/ticket-detail.png
  - Admin panel (territories or metrics)                → assets/admin.png

Reference them with:
  ![Intake screen](assets/intake.png)
-->

---

## Architecture

For a full walkthrough of the system design — the data model, routing logic, role hierarchy, and key technical decisions — see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Development Methodology

This project was built using an AI-collaborative development process. See [METHODOLOGY.md](METHODOLOGY.md) for how that worked.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js (App Router), React, TypeScript |
| Styling | Tailwind CSS |
| Backend | Supabase (PostgreSQL + Auth + Row-Level Security) |
| Deployment | Vercel |

No custom backend server. Supabase's auto-generated REST API handles all data access, with Row-Level Security policies enforcing permissions at the database layer. Business logic lives in the database, not in application code.

---

*This is the public landing page for a private repository.*
