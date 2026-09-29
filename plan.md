# HKSA Website: Architecture & Plan

Website for the **Hong Kong Student Association (HKSA) at UCSB**. The site has:
- a photo gallery
- the latest Instagram posts
- a family tree that admins can edit
- a suggestion form for feedback, event ideas and collab ideas
- an events calendar with RSVP
- an officer board page
- English / 繁體中文 support

## 1. Key decisions

| Area | Decision | Why |
|---|---|---|
| Framework | Next.js 16 App Router (React 19, TypeScript, Tailwind 4) | Already set up. Server Components + Server Actions mean no separate backend is needed. |
| Backend | **Supabase**: Postgres + Storage + Auth | One free-tier service for the database, photo files and admin login. Easy to hand off to next year's officers. |
| ORM | Drizzle ORM | Type-safe SQL, migrations kept in the repo, works well with Supabase Postgres. |
| Hosting | Vercel (free tier) + Vercel Cron | Made for Next.js, with preview deploys for every PR. |
| Photos | Files in **Supabase Storage**, metadata in **Postgres** | Storing image bytes in Postgres bloats the DB and skips the CDN. The DB stores paths, captions and albums only. |
| Image processing | **Resize in the browser at upload time** (thumbnail + display size, HEIC → JPEG/WebP). Serve from Storage **without** Vercel image optimization. | Vercel Hobby has a limited monthly image-optimization quota, and a gallery of hundreds of photos would use it up. iPhone photos are often HEIC. |
| Instagram | **Behold.so feed now**, official Instagram API later | The old Basic Display API was shut down in Dec 2024. The official API needs a Business/Creator account plus a token refreshed every 60 days. A provider interface lets us swap later. |
| Suggestion form | One form, three categories: **feedback**, **event suggestion**, **collab suggestion** | A single place for anyone to send ideas. Officers review them in admin. See §6. |
| Join page | **Out of scope for this plan**: Kelvin builds `/join` by hand | Shoreline has no public write API, so the site won't integrate with it. |
| Family tree | Named families. **Each little has exactly one big and inherits the big's family.** | Matches how HKSA families work. Only founders are assigned to a family directly. See §4. |
| Member privacy | Members are **visible by default**. Admins hide anyone who asks. | Keeps the tree complete. The admin UI provides a quick hide toggle. |
| Admin access | Google login + `admins` email allowlist, **not** restricted to `@ucsb.edu`. The first admin is seeded with SQL, and admins manage the list at `/admin/admins`. | Officers may use personal Gmail accounts. Handing off each year needs no code change. |
| Styling | **Deferred.** Kelvin + the publicity team are designing the UI/UX. Build functional, layout-only pages with design tokens. See §7. | Lets features be built now without committing to a look, and keeps the future restyle cheap. |
| Supabase free-tier pause | Daily `/api/cron/keepalive` job runs a trivial query | Free projects pause after about 7 days without activity (e.g. summer or winter break). |

## 2. Tech stack

- **UI:** Tailwind 4, shadcn/ui components, `yet-another-react-lightbox` for the gallery, `@xyflow/react` (React Flow) for the family tree
- **Data:** Supabase Postgres via Drizzle (server-side only), with Row Level Security on for all tables
  - **RLS stance:** Drizzle connects as the postgres role, which **bypasses RLS**. The real authorization check is `isAdmin()` in every admin Server Action and route handler.
  - RLS is on for every table with **no anon policies**, so the tables can't be reached with the public anon key.
- **Files:** a Supabase Storage bucket `photos` with public read.
  - The browser resizes each image (thumbnail + display size, converting HEIC), then uploads directly with a signed upload URL.
  - Gallery images are served straight from Storage (`next/image` with `unoptimized` or a custom loader), not through Vercel image optimization.
- **Auth:** Supabase Auth (Google login) via `@supabase/ssr`. Only emails in the `admins` table can access `/admin`.
- **Validation:** Zod on every form and Server Action
- **i18n:** an `app/[lang]/` route segment (`en`, `zh-Hant`) with JSON dictionaries, following Next's built-in internationalization guide. DB content has optional `_zh` columns.
- **Spam protection:** Cloudflare Turnstile on public forms
- **Email (optional):** Resend, for RSVP confirmations and "new suggestion" alerts to officers

## 3. Architecture overview

```
Browser ──> Vercel (Next.js 16)
              ├─ app/[lang]/*        public pages (Server Components, cached)
              ├─ app/admin/*         admin UI (guarded by proxy.ts + isAdmin())
              ├─ Server Actions      mutations (Zod-validated, admin re-checked)
              ├─ app/api/*           cron + CSV export route handlers
              ├─ components/*        presentational components (props in, no DB)
              │
              ├──> Supabase Postgres (via Drizzle)
              ├──> Supabase Storage  (photos bucket: display + thumbnail files)
              ├──> Supabase Auth     (admin login)
              └──> Behold.so / Instagram API (cached ~1h)
```

## 4. Data model (Postgres)

- `albums` (id, slug, title, title_zh, event_date, cover_photo_id, **event_id?**): `event_id` is a nullable FK to `events`, so an event page can show its album
- `photos` (id, album_id, storage_path, **thumb_path**, width, height, caption, caption_zh, taken_at, sort_order)
- `families` (id, name, name_zh, color, founded_year)
- `members` (id, name, chinese_name, grad_year, family_id, photo_path, is_visible **default true**)
- `bigs_littles` (big_id, little_id, year): lineage edges
- `suggestions`:
  - `id`, `created_at`
  - `category`: enum of `feedback` | `event` | `collab`
  - `message`
  - `name?`, `email?`: optional; the form can be sent anonymously
  - category-specific fields, all optional:
    - event: `event_title`, `proposed_date`
    - collab: `org_name`, `contact`
  - `status`: enum of `new` | `reviewing` | `planned` | `done` | `archived`
  - `admin_notes`
- `events` (id, slug, title, title_zh, description, starts_at, ends_at, location, cover_path, rsvp_open, capacity)
- `rsvps` (event_id, name, email, created_at), unique on (event_id, email). Emails are visible to admins only.
- `officers` (id, name, role, role_zh, bio, photo_path, term such as `"2026-27"`, sort_order)
- `admins` (email)
- `instagram_cache` (id, payload jsonb, fetched_at), used once the official API is in place

### Family tree rules
- **One big per little:** `bigs_littles.little_id` is unique.
- **Family is inherited:** `members.family_id` is set **only on founders** (members with no big). Everyone else's family is found by walking up the lineage to the root. The tree is small, so this is done in app code.
- Admin Server Actions enforce:
  - A member with a big has `family_id = null`.
  - No edge may create a cycle (the new big can't be a descendant of the little).
  - Deleting or re-parenting a member leaves the tree valid (e.g. orphaned littles become founders or are reassigned).

## 5. Routes

**Public pages** (`app/[lang]/`):
- `/`: hero, next upcoming event, Instagram strip
- `/gallery`, `/gallery/[album]`: albums and a photo grid with a lightbox
- `/family-tree`: React Flow tree, filterable by family
- `/events`, `/events/[slug]`: event list and details, RSVP form, "add to calendar" `.ics` file, linked album if one exists
- `/board`: current officers, plus an archive of past boards
- `/suggest`: suggestion form. `?type=event|collab|feedback` preselects the category, so other pages (such as events) can link straight to it.
- `/join`: built by Kelvin (not part of this plan)

**Admin pages** (`app/admin/`): `proxy.ts` guards them, and every Server Action also calls `isAdmin()`.
- Dashboard
- Albums & photos: upload (with browser-side resizing), reorder, caption, link to an event
- Families, members (with a quick hide toggle) and the big/little lineage editor
- Events and RSVP lists
- Officers
- Suggestions inbox: filter by category and status, change status, add notes, CSV export
- Admins: add or remove admin emails

**Route handlers:**
- `/api/cron/keepalive`: daily Vercel Cron job (`select 1`) so Supabase doesn't pause. Protected by `CRON_SECRET`.
- `/api/cron/instagram-refresh`: Vercel Cron job that refreshes the token and cache (for the official API later)
- `/api/export/suggestions.csv` (admin only)

## 6. Suggestion form

One form at `/suggest` handles **feedback on anything**, **event suggestions** and **collab suggestions**.

- **Category picker** at the top (Feedback / Event idea / Collab idea). The fields below change with the category:
  - Feedback: message
  - Event idea: event title, rough date or timeframe, description
  - Collab idea: org or business name, contact info (optional), description
- **Name and email are optional.** Anonymous feedback is allowed; people who want a reply leave an email.
- **Server Action** that validates with a Zod discriminated union keyed on `category`, verifies Turnstile, and inserts into `suggestions`.
- **Abuse limits:** length caps on text fields, plus a simple per-IP rate limit (a small table or Upstash, if needed).
- **Admin inbox** at `/admin/suggestions`: filter by category and status, move items through `new → reviewing → planned → done`, add private notes, export CSV.
- **Optional:** a Resend email to officers when a new suggestion comes in.

## 7. UI/UX approach (styling deferred)

Kelvin and the club's publicity team are designing the visual design separately. Until they're done, pages are **functional and unstyled**, built so the restyle is cheap:

- **Semantic, accessible HTML:** labels, headings, landmarks, alt text. No decorative styling.
- **Tailwind for layout only:** flex/grid, spacing, responsive breakpoints. Mobile-first.
- **Design tokens:** colors, fonts and radii are defined once as `@theme` tokens in `app/globals.css` (e.g. `--color-brand`), using neutral placeholders for now, including a CJK font fallback. The brand drops in by editing that one file.
- **Separate data from display:**
  - Pages and Server Actions fetch and mutate data.
  - Presentational components in `components/` only receive typed props and never touch the DB.
  - A redesign should only touch `components/` and `globals.css`.
- **shadcn/ui primitives** for form controls, dialogs and selects (accessible behavior). They restyle through the same tokens.
- Allow for en and zh-Hant strings being different lengths.

## 8. Instagram design

`lib/instagram/` exports `getLatestPosts(n)`, backed by an `InstagramProvider` interface:
- `BeholdProvider` (now): fetches a Behold.so JSON feed URL (`BEHOLD_FEED_URL`).
- `GraphProvider` (later): the official Instagram API with Instagram Login. It needs the account switched to **Business/Creator** and a long-lived token (60 days) refreshed by a Vercel Cron job.

An environment variable (`INSTAGRAM_PROVIDER`) chooses the provider. The result is cached for about an hour with Next's caching/revalidation. If the feed fails, the page falls back to a "Follow us on Instagram" link.

## 9. Phased roadmap

The whole plan is built **in order, with no rush to launch**. Design and architecture are agreed before each phase is implemented.

1. **Foundation**:
   - Supabase project, Drizzle schema and migrations (RLS enabled on every table), `.env` setup with Zod-validated env
   - `[lang]` layout and dictionaries
   - unstyled nav/footer shell with placeholder pages
   - `@theme` placeholder tokens
   - keepalive cron
2. **Admin auth**: Supabase Auth (Google), `admins` allowlist, `proxy.ts` guard, admin shell, `/admin/admins` management
3. **Gallery**: browser-side resizing + upload to Storage, album pages (optionally linked to events), lightbox
4. **Suggestion form**: `/suggest` with category-specific fields, Zod + Turnstile, saves to `suggestions`, admin inbox + CSV export
5. **Family tree**: families/members/lineage admin CRUD (create, read, update, delete) with the invariants in §4 and a hide toggle, plus a React Flow public view
6. **Events + RSVP**: list and detail pages, RSVP, `.ics` download
7. **Officer board**: current board plus archive
8. **Instagram**: Behold feed first, then the Graph API and token-refresh cron
9. **Polish**:
   - apply the final visual design from publicity
   - 繁中 translations
   - SEO metadata and OG images
   - accessibility pass
   - Vercel Analytics

## 10. Environment variables (planned)

```
DATABASE_URL=                     # Supabase Postgres (pooled)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=        # server only
INSTAGRAM_PROVIDER=behold         # behold | graph
BEHOLD_FEED_URL=
INSTAGRAM_ACCESS_TOKEN=           # later, graph provider
TURNSTILE_SITE_KEY= / TURNSTILE_SECRET_KEY=
CRON_SECRET=                      # protects /api/cron/* (keepalive, instagram-refresh)
```

## 11. Open items

- **Visual design** (colors, logo usage, typography, tone): being designed by Kelvin + publicity.
- **Family tree layout:** top-down vs. sideways, one family at a time vs. all families, mobile behavior. Decide with publicity before Phase 5.
- **Gallery grid style:** uniform grid vs. masonry. This affects thumbnail sizes, so decide before Phase 3.
- **RSVP details:** is a waitlist needed once capacity is reached? Can people cancel their own RSVP? Decide before Phase 6.

## 12. Future feature ideas

- Public "You asked, we did" board showing suggestions marked `done`
- Resend email newsletter
- Merch / dues page (Stripe payment links)
- Alumni directory (opt-in)
- Cantonese word of the week / culture blog
- Sponsor and partner org showcase
- Event check-in via QR code for attendance points
