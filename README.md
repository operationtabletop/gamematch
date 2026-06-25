# GameMatch — Operation Tabletop Game Night Group Matcher

Automatically groups game-night participants who want to play the same game and
emails each group so they can connect. Registrations come from the existing
Operation Tabletop **Monday.com** form — no change to how players sign up.

Today this matching is done by hand over email. GameMatch automates it.

## What it does

1. Staff opens GameMatch and picks an **event date** (and optionally a location:
   Hurlburt Field or Eglin).
2. The app pulls that day's registrations **live from Monday.com**.
3. It keeps everyone who answered **"Yes"** to *"Are you interested in finding a
   group?"* and groups them by the game they want to play.
4. Staff reviews the groups and clicks **Email this group** (or **Send all
   emails**) to notify every player in a group with their group's details.

No database to maintain — **Monday.com stays the single source of truth.**

## Status

📋 Planning complete. The full build spec and step-by-step directions live in
[`docs/BUILD_PLAN.md`](docs/BUILD_PLAN.md). Nothing is built yet.

## Recommended stack (summary)

| Layer | Choice | Why |
|-------|--------|-----|
| App framework | **Next.js (App Router) + TypeScript** | UI + secure server API in one deployable project |
| UI | **Tailwind CSS + shadcn/ui** | Clean, fast to build, accessible |
| Data source | **Monday.com GraphQL API** | Already where registrations live; no separate DB |
| Email | **Resend** | Simple API, generous free tier, great deliverability |
| Hosting | **Vercel** | Push-to-deploy from GitHub, free tier covers an internal tool |
| Access control | **Single shared passcode** (env var) | Staff-only internal tool; upgrade to real auth later |

See [`docs/BUILD_PLAN.md`](docs/BUILD_PLAN.md) for the full rationale, architecture,
and build steps.

## What's needed to build it

- A **Monday.com API token** (read access to the registration board)
- The **board ID** of the registration board
- A **Resend account** + verified sending domain (e.g. `@operationtabletop.org`)
- A **Vercel account** connected to this GitHub repo

Details and exact column IDs are in the build plan.
