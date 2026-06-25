# GameMatch — Operation Tabletop Game Night Group Matcher

A standalone web app for Operation Tabletop game nights. Players register for an
event and say which game they want a group for; GameMatch groups everyone who
wants the same game and notifies them — **in the app and by email**.

This replaces the manual email matching done today. **No Monday.com** — GameMatch
owns the whole flow: registration, data, matching, and notifications.

## How it works

**Players**
1. Create an account (email/password or Google) and sign in.
2. Browse upcoming game nights and register, choosing the game they want a group
   for and confirming they're looking for a group.
3. Once staff run the match, players see **their group and groupmates in the app**
   and get an **email** with the details.

**Staff (admin, behind a shared passcode)**
1. Create/edit game-night events (date, location).
2. View registrations for an event.
3. **Run matching** — groups everyone interested by their requested game (no size
   limit; connect everyone who wants that game).
4. **Send emails** to each group (or all groups at once).

## Recommended stack (summary)

| Layer | Choice | Why |
|-------|--------|-----|
| Framework | **Next.js (App Router) + TypeScript** | UI + secure server API in one deployable app |
| UI | **Tailwind CSS + shadcn/ui** | Clean, fast, accessible |
| Database | **PostgreSQL** (Neon or Vercel Postgres) | Reliable relational store for users, events, registrations, groups |
| ORM | **Prisma** | Type-safe schema & queries, easy migrations |
| Player auth | **Clerk** | Email/password + Google, verification & resets out of the box |
| Admin auth | **Shared passcode** (middleware) | Simple staff gate for v1 |
| Email | **Resend** | Simple API, generous free tier, good deliverability |
| Hosting | **Vercel** | Push-to-deploy from GitHub; free tier covers an internal tool |

Full rationale, data model, and step-by-step directions:
[`docs/BUILD_PLAN.md`](docs/BUILD_PLAN.md).

## Status

📋 Planning complete; nothing built yet. See the build plan for next steps and the
short list of accounts/keys needed (Postgres, Clerk, Resend, Vercel).
