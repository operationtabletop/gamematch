# GameMatch — Complete Build Plan & Directions

> Self-contained directions for building the Operation Tabletop Game Night Group
> Matcher. Written so a future session (or developer) can build it end-to-end
> without re-deriving the requirements. Read this top-to-bottom before coding.

---

## 1. The goal

Operation Tabletop runs game nights at military community locations (Hurlburt
Field, Eglin). Participants register through a **Monday.com form**. Staff
currently match players into groups by game preference **manually over email**.

GameMatch automates the matching and notification:

- Read registrations **live from Monday.com** for a chosen event date/location.
- Keep everyone who said **Yes** to *"Are you interested in finding a group?"*.
- **Group them by the game** they want to play (no group-size limit — connect
  *everyone* interested in a given game).
- Let staff **email each group** so players can connect.

Non-goals for v1: changing the registration flow, player-facing login, payments,
scheduling, chat. Monday.com remains the system of record.

---

## 2. Recommended tech stack

**Build it as a single Next.js app deployed on Vercel.** One repo, one deploy,
secure server-side secrets, no infrastructure to babysit.

| Concern | Recommendation | Notes / alternatives |
|--------|----------------|----------------------|
| Framework | **Next.js (App Router) + TypeScript** | UI pages + server-side API routes in one project. Server routes keep the Monday.com token and Resend key off the client. |
| Styling/UI | **Tailwind CSS + shadcn/ui** | Fast, clean, accessible components (buttons, cards, date picker, toasts). |
| Data | **Monday.com GraphQL API v2** | https://developer.monday.com/api-reference. No separate database needed for v1. |
| Email | **Resend** (`resend` npm pkg) | 3,000 emails/mo free, simple API, good deliverability. Alternatives: SendGrid, Mailgun, AWS SES. |
| Hosting | **Vercel** | Connect this GitHub repo → auto-deploy on push. Free "Hobby" tier is enough for an internal tool. |
| Access control | **Single shared passcode** via Next.js middleware (env var) | Staff-only. Upgrade path: Auth.js with Google sign-in restricted to `@operationtabletop.org`. |
| State / DB | **None for v1** | Monday.com is the source of truth. *Optional:* track "already emailed" by writing back to a Monday.com column instead of adding a DB. |

**Why not just keep it as a Claude artifact?** The artifact works for ad-hoc use
inside a Claude chat, but a deployed app gives you: a stable URL staff can
bookmark, secrets handled securely server-side, reliable scheduled email sending,
and no dependency on opening a Claude conversation each time.

**Why no database?** The only "state" is the registration data (lives in
Monday.com) and optionally "who has been emailed" (can be a checkbox column
written back to Monday.com). Adding Postgres/etc. would be overkill for v1.

---

## 3. Architecture

```
Browser (staff)                Next.js on Vercel                 External
─────────────────             ───────────────────               ──────────
[ /  matcher UI ] ── fetch ──► [ /api/registrations ] ─ GraphQL ► Monday.com API
   pick date/loc                  (server, holds token)
   see groups
   click "Email" ─── POST ────► [ /api/send-emails ]  ─ REST ───► Resend API
                                   (server, holds key)
[ middleware: passcode gate on all routes ]
```

- **Client** never sees the Monday.com token or Resend key — both live only in
  server-side API routes as environment variables.
- **Matching logic** runs server-side (or client-side over already-fetched data;
  either is fine since it's simple grouping). Keep it in a shared `lib/` module.

---

## 4. Monday.com data model (known column IDs)

From inspecting the live registration board, these are the relevant columns.
**Confirm the board ID and re-verify these IDs before building** (IDs are stable
but the board may evolve).

| Field | Column ID | Type | Use |
|-------|-----------|------|-----|
| Full name | `name` | item name | Display in groups & email greeting |
| Email | `emailrryi8rcj` | email | Recipient address |
| Interested in finding a group? | `single_selectzk6zhjb` | status/single-select | **Filter: keep only "Yes"** |
| Game wanted | `short_text1j90x04o` | short text | **Group key** |
| Event date | `dates7uh919k` | date | Filter to the chosen event |
| Location | *(TBD — confirm column ID)* | status/text | Optional filter: Hurlburt Field / Eglin |

> ⚠️ **Action item:** confirm the **board ID** and the **location column ID** via
> the Monday.com API (`boards { columns { id title type } }`) before coding. The
> column IDs above came from a prior inspection and should be re-verified.

### Example GraphQL query

```graphql
query ($boardId: ID!) {
  boards(ids: [$boardId]) {
    items_page(limit: 500) {
      cursor
      items {
        id
        name
        column_values {
          id
          text
          value
        }
      }
    }
  }
}
```

Paginate with `cursor` if a board exceeds the page limit. Parse each item's
`column_values` by `id` into a typed `Registration` object.

---

## 5. Matching logic

```
Registration = { id, name, email, game, eventDate, location, interested }

1. Fetch all items for the board.
2. Filter:
     - interested === "Yes"
     - eventDate === selectedDate
     - (optional) location === selectedLocation
3. Normalize the game string (trim, lowercase for the key, but keep a
   display label). Optionally collapse obvious variants
   (e.g. "Catan" / "Settlers of Catan") — start simple, refine later.
4. Group by normalized game key.
5. Output: Group = { game, players: Registration[] }, sorted by player count desc.
```

Keep group size **unlimited** — the goal is to connect everyone interested in a
game, not cap groups. Show a count per group in the UI.

Edge cases to handle: empty game field (skip or bucket as "Unspecified"),
duplicate registrations (dedupe by email within a game), missing email (flag in
UI, don't crash sending).

---

## 6. Screens / UI

Single page is enough for v1:

1. **Controls bar:** event date picker (defaults to today), optional location
   filter, **Load Registrations** button.
2. **Groups list:** one card per game, showing the game name, player count, and
   the list of players (name + email). Each card has **Email this group**.
3. **Global action:** **Send all emails** button + a summary (X groups, Y
   players, Z with missing emails).
4. **Feedback:** loading state while fetching, success/error toasts after
   sending, and a per-group "Emailed ✓" indicator.

### Email content (per group)

- **To:** each player in the group (or BCC the group, From a no-reply Operation
  Tabletop address).
- **Subject:** `Your Game Night group for {game} — {eventDate}`
- **Body:** friendly note listing everyone in the group (names + emails) so they
  can coordinate, plus event date/location. Keep a templated, editable body in a
  `lib/emailTemplate.ts`.

---

## 7. Step-by-step build directions

1. **Scaffold:** `npx create-next-app@latest` (TypeScript, App Router, Tailwind).
   Add shadcn/ui. Commit.
2. **Env & config:** create `.env.local` with the variables in §8; add
   `.env.example` (no secrets) to the repo; ensure `.env.local` is gitignored.
3. **Monday.com client** (`lib/monday.ts`): a `fetchRegistrations()` that runs
   the GraphQL query, paginates, and maps `column_values` → `Registration[]`.
   Read `MONDAY_API_TOKEN` and `MONDAY_BOARD_ID` from env.
4. **Verify board schema first:** write a tiny script/route that prints
   `columns { id title type }` so you can confirm the board ID, location column,
   and the IDs in §4 against the live board before wiring the rest.
5. **Matching** (`lib/match.ts`): pure function implementing §5. Unit-test it
   with sample data.
6. **API routes:**
   - `GET /api/registrations?date=&location=` → fetch + filter + group, return JSON.
   - `POST /api/send-emails` (body: a group or "all") → send via Resend.
7. **UI** (`app/page.tsx`): controls bar, groups list, email buttons, toasts —
   per §6.
8. **Email** (`lib/email.ts`): Resend client + `emailTemplate.ts`. Send to a test
   inbox first.
9. **Access gate** (`middleware.ts`): redirect to a passcode prompt unless a
   cookie/header matches `APP_PASSCODE`. Keep it simple.
10. **Polish:** error states, missing-email handling, "Emailed ✓" indicator,
    empty-state messaging.
11. **Deploy:** push to GitHub → import into Vercel → set env vars in Vercel →
    verify the production URL → share with staff.
12. *(Optional)* **Track sent state:** add a checkbox/status column in Monday.com
    and have `/api/send-emails` write back so re-runs don't double-email.

---

## 8. Environment variables / secrets

```
MONDAY_API_TOKEN=        # Monday.com API v2 token, read access to the board
MONDAY_BOARD_ID=         # numeric board ID of the registration board
RESEND_API_KEY=          # Resend API key
EMAIL_FROM=Operation Tabletop <gamenight@operationtabletop.org>  # verified sender
APP_PASSCODE=            # shared staff passcode for the access gate
```

Never commit real values. Set them in `.env.local` for dev and in the Vercel
project settings for production. Commit a `.env.example` with the keys and empty
values.

---

## 9. What I need from Erica to build this

- [ ] **Monday.com API token** — *Monday.com → Avatar → Developers → My Access
      Tokens → copy.* (Or an admin can scope a token to read the board.)
- [ ] **Board ID** of the registration board (the number in the board's URL).
- [ ] **Resend account** + a **verified sending domain/address** (ideally on
      `operationtabletop.org` for deliverability). I'll provide DNS records if
      needed.
- [ ] **Vercel account** with access to connect the `operationtabletop/gamematch`
      GitHub repo.
- [ ] Confirmation of the **location column** and whether v1 needs the location
      filter or can ship date-only first.

I can build and commit the full app structure first (with the schema-verification
step), then plug in your token/board ID to wire it to live data.

---

## 10. Future enhancements (post-v1)

- Real auth (Auth.js + Google, restricted to `@operationtabletop.org`).
- Smarter game matching (fuzzy matching of game-name variants/aliases).
- Track emailed/contacted state back in Monday.com (no double-emails).
- Per-event history & simple analytics (turnout by game).
- Optional group-size targets / splitting large groups.
- Scheduled auto-run + auto-email at a set time before each event.
- Player replies / RSVP confirmation loop.

---

## 11. Open questions

1. Should emails go **to all players directly** (everyone sees each other's
   email to self-organize) or **individually/BCC** for privacy? (Recommend asking
   Erica — affects the template and To/BCC handling.)
2. Is there a **location column**, and is the location filter needed for v1?
3. Any **branding** (logo, colors) to apply to the UI and email template?
4. Should re-running for the same event **avoid re-emailing** people already
   contacted? (Drives the optional Monday.com write-back.)
