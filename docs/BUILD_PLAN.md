# GameMatch — Complete Build Plan & Directions

> Self-contained directions for building the Operation Tabletop Game Night Group
> Matcher as a **standalone web application** (no Monday.com). Written so a future
> session or developer can build it end-to-end. Read top-to-bottom before coding.

---

## 1. The goal

Operation Tabletop runs game nights (locations include Hurlburt Field and Eglin).
Players want to find groups to play specific games. Today staff match players by
hand over email. GameMatch is a self-contained web app that:

- Lets **players register** for an event and pick the game they want a group for.
- **Groups** everyone who wants the same game at that event (no size cap — connect
  *everyone* interested in a game).
- Notifies matched players **in the app and by email**.
- Gives **staff** a passcode-protected admin to create events, run matching, and
  send emails.

### Scope decisions (confirmed)

| Decision | Choice |
|----------|--------|
| Player identity | **Accounts with login** (email/password or Google) |
| Match delivery | **Email + in-app** (players see their group in the app) |
| Admin access | **Single shared staff passcode** for v1 |
| Data source | **The app's own database** (Monday.com removed entirely) |

Non-goals for v1: payments, chat, mobile native apps, per-user staff roles
(single shared admin passcode is enough to start).

---

## 2. Recommended tech stack

**One Next.js app, one Postgres database, deployed on Vercel.**

| Concern | Recommendation | Notes / alternatives |
|--------|----------------|----------------------|
| Framework | **Next.js (App Router) + TypeScript** | UI pages + server API/Server Actions in one project; secrets stay server-side. |
| Styling/UI | **Tailwind CSS + shadcn/ui** | Buttons, cards, dialogs, date picker, toasts — fast and accessible. |
| Database | **PostgreSQL** | Use **Neon** (serverless, generous free tier) or **Vercel Postgres**. |
| ORM | **Prisma** | Type-safe models, easy migrations. (Alt: Drizzle.) |
| Player auth | **Clerk** | Email/password **and** Google, plus email verification & password reset with almost no code. Free tier covers thousands of users. (Free alt: **Auth.js / NextAuth v5** — more wiring for credentials + verification.) |
| Admin auth | **Shared passcode** via middleware on `/admin/*` | Staff-only. Upgrade path: give staff Clerk accounts with an `admin` role and drop the passcode. |
| Email | **Resend** (`resend` pkg) | 3,000 emails/mo free, simple API. (Alts: SendGrid, Mailgun, SES.) |
| Hosting | **Vercel** | Connect this GitHub repo → auto-deploy on push. Free Hobby tier is enough. |

**Consolidated alternative:** **Supabase** gives Postgres + Auth + storage from one
vendor. Fewer accounts to manage, but Clerk's player auth UX is more turnkey. The
plan below assumes **Clerk + Postgres + Prisma**; swapping to Supabase changes only
the auth/DB wiring, not the data model or features.

**Why a real app (vs. the earlier Monday.com artifact):** a standalone app gives
you player accounts, in-app group viewing, your own data you control, a stable URL,
and reliable scheduled email — none of which depend on a third-party board.

---

## 3. Architecture

```
                         Next.js on Vercel
Browser (player) ──┐    ┌───────────────────────────┐        External
  sign in (Clerk)  ├──► │ Player pages:             │
  browse events    │    │  /, /events, /events/[id],│
  register         │    │  /me (my groups)          │
  see my group     │    │                           │
                   │    │ Server Actions / API:     │ ── SQL ──► PostgreSQL
Browser (staff) ───┤    │  register, run-match,     │            (Prisma)
  /admin (passcode)│    │  send-emails              │
  create events    │    │                           │ ── REST ─► Resend (email)
  run match / email└──► │ middleware: Clerk auth +  │
                        │ passcode gate on /admin   │ ◄─ auth ── Clerk
                        └───────────────────────────┘
```

- **Players** authenticate via Clerk. **Admin** routes additionally require the
  shared passcode (cookie set after entering it).
- DB access and email sending happen only server-side (Server Actions or route
  handlers). Secrets never reach the client.
- Matching is a pure server-side function over registrations for an event.

---

## 4. Data model (Prisma schema sketch)

```prisma
model User {            // mirror of the Clerk user we care about
  id            String   @id              // Clerk user id
  email         String   @unique
  fullName      String?
  createdAt     DateTime @default(now())
  registrations Registration[]
}

model Event {
  id            String   @id @default(cuid())
  title         String
  date          DateTime
  location      String?                    // e.g. "Hurlburt Field", "Eglin"
  status        EventStatus @default(UPCOMING)
  createdAt     DateTime @default(now())
  registrations Registration[]
  groups        Group[]
}

enum EventStatus { UPCOMING MATCHED CLOSED }

model Registration {
  id           String   @id @default(cuid())
  user         User     @relation(fields: [userId], references: [id])
  userId       String
  event        Event    @relation(fields: [eventId], references: [id])
  eventId      String
  game         String                       // the game they want a group for
  lookingForGroup Boolean @default(true)
  groupMember  GroupMember?
  createdAt    DateTime @default(now())

  @@unique([userId, eventId, game])         // one reg per game per event
  @@index([eventId])
}

model Group {
  id        String   @id @default(cuid())
  event     Event    @relation(fields: [eventId], references: [id])
  eventId   String
  game      String
  emailedAt DateTime?                        // null until notified
  members   GroupMember[]
  createdAt DateTime @default(now())
  @@index([eventId])
}

model GroupMember {
  id             String       @id @default(cuid())
  group          Group        @relation(fields: [groupId], references: [id])
  groupId        String
  registration   Registration @relation(fields: [registrationId], references: [id])
  registrationId String       @unique
}
```

Notes:
- A player may register for **multiple games** at one event (one `Registration`
  each) — the unique key allows it.
- `Group`/`GroupMember` are produced by the matching step so players can view
  their group in-app and so emails can be tracked (`emailedAt`).

---

## 5. Matching logic

```
For a given event:
  1. Load registrations where eventId = event AND lookingForGroup = true.
  2. Normalize game name -> key (trim; lowercase for the key; keep a display label).
     Optionally collapse known aliases (e.g. "Catan" == "Settlers of Catan").
  3. Group registrations by game key.
  4. For each game with >= 1 interested player, create/replace a Group and its
     GroupMembers. (Re-running replaces prior groups for that event.)
  5. Set event.status = MATCHED.
```

Unlimited group size by design. Edge cases: empty game string (skip or bucket as
"Unspecified"); a player registering the same game twice (prevented by unique key);
re-running matching should be idempotent (clear old groups for the event first).

---

## 6. Screens / UI

### Player-facing (Clerk-authenticated)
- **Sign in / sign up** — Clerk components (email/password + Google).
- **Home / Events** (`/`, `/events`) — list of upcoming game nights.
- **Event detail** (`/events/[id]`) — event info + a **Register** action: choose
  the game (free text, with suggestions of games others picked for this event) and
  confirm "looking for a group". Shows the player's current registrations for the
  event; allow edit/cancel before matching.
- **My groups** (`/me`) — once an event is matched, show the player's group(s):
  game, groupmates (names + emails so they can coordinate), event date/location.

### Admin (passcode-gated, `/admin`)
- **Passcode prompt** → sets a cookie.
- **Events list** — create / edit / close events.
- **Event admin** (`/admin/events/[id]`) — see all registrations; **Run matching**
  → preview groups (game + members + counts); **Send emails** per group or **Send
  all**; shows `emailedAt` status per group.

### Emails
- **Subject:** `Your Game Night group for {game} — {date}`
- **Body:** friendly note listing groupmates (names + emails) + event date/location
  + a link to `/me`. Template lives in `lib/emailTemplate.ts`.
- Decide To-vs-BCC per the privacy question in §11.

---

## 7. Step-by-step build directions

1. **Scaffold:** `npx create-next-app@latest` (TypeScript, App Router, Tailwind,
   ESLint). Add shadcn/ui. Commit.
2. **Database:** create a Neon (or Vercel) Postgres DB. Add `DATABASE_URL` to env.
   `npm i prisma @prisma/client`; `npx prisma init`; add the §4 schema;
   `npx prisma migrate dev`.
3. **Player auth (Clerk):** create a Clerk app; add keys to env; wrap the app in
   `<ClerkProvider>`; add sign-in/up routes; protect player pages in `middleware.ts`.
   On first sign-in, upsert the `User` row (Clerk webhook or on-request upsert).
4. **Admin gate:** in `middleware.ts`, additionally require a passcode cookie for
   `/admin/*`; build a small passcode form that sets the cookie when `APP_PASSCODE`
   matches.
5. **Events:** admin pages + Server Actions to create/edit/close events; public
   events list + detail pages.
6. **Registration:** Server Action to create/edit/cancel a `Registration` for the
   signed-in user on an event (game + lookingForGroup). Validate input.
7. **Matching** (`lib/match.ts`): pure function per §5; a Server Action that clears
   old groups for the event, writes new `Group`/`GroupMember` rows, sets status
   MATCHED. Unit-test the pure function.
8. **In-app group view:** `/me` reads the player's groups + groupmates.
9. **Email** (`lib/email.ts` + `emailTemplate.ts`): Resend client; Server Action to
   send per-group or all; set `emailedAt`. Test to a real inbox first.
10. **Polish:** empty states, loading/toasts, missing-email handling, idempotent
    re-match, "Emailed ✓" indicators.
11. **Deploy:** push to GitHub → import to Vercel → set env vars (DB, Clerk, Resend,
    passcode) → run prod migration → verify → share URL with players/staff.
12. *(Optional)* Replace the admin passcode with Clerk roles; add scheduled
    auto-match/auto-email before each event.

---

## 8. Environment variables / secrets

```
DATABASE_URL=                              # Postgres connection string (Neon/Vercel)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=         # Clerk
CLERK_SECRET_KEY=                          # Clerk
RESEND_API_KEY=                            # Resend
EMAIL_FROM=Operation Tabletop <gamenight@operationtabletop.org>  # verified sender
APP_PASSCODE=                              # shared staff admin passcode
```

Keep real values out of git. Use `.env.local` in dev and Vercel project settings in
prod. Commit a `.env.example` with empty values.

---

## 9. What I need from Erica to build & ship this

- [ ] **Postgres database** — create a free **Neon** project (or use Vercel
      Postgres) and share the `DATABASE_URL`. (I can scaffold against a local DB
      first and you plug this in at deploy.)
- [ ] **Clerk account** — create an app at clerk.com; share the publishable +
      secret keys; decide if Google sign-in should be enabled (recommended).
- [ ] **Resend account** + **verified sending domain/address** (ideally on
      `operationtabletop.org` for deliverability). I'll provide the DNS records.
- [ ] **Vercel account** with access to connect `operationtabletop/gamematch`.
- [ ] A **staff passcode** to set as `APP_PASSCODE`.

I can build the entire app structure, data model, matching, and UI **before** any
keys exist (using a local Postgres + Clerk dev keys), then you fill in production
values at deploy time.

---

## 10. Future enhancements (post-v1)

- Per-user staff logins / roles (drop the shared passcode).
- Smarter game matching (fuzzy alias matching, autocomplete from a games catalog).
- Scheduled auto-match + auto-email a set time before each event.
- RSVP / confirmation loop and reminders.
- Player profiles, favorite games, attendance history, simple analytics.
- Mobile-friendly PWA / native wrapper.

---

## 11. Open questions

1. **Email privacy:** should group emails show everyone's address (so players
   self-organize) or be sent individually/BCC? (Affects To/BCC + template.)
2. **Locations:** fixed list (Hurlburt Field, Eglin) or free-form per event?
3. **Branding:** logo/colors for the UI and email template?
4. **Google sign-in:** enable it in Clerk in addition to email/password?
5. **Multiple games per player per event:** confirmed allowed — OK to keep?
