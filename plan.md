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
| Instagram | **Behold.so feed now**, official Instagram API later | The old Basic Display API was shut down in Dec 2024. The official API needs a Business/Creator account plus a token refreshed every 60 days. A provider interface lets us swap later. |
| Suggestion form | One form, three categories: **feedback**, **event suggestion**, **collab suggestion** | A single place for anyone to send ideas. Officers review them in admin. See §6. |
| Join page | **Out of scope for this plan**: Kelvin builds `/join` by hand | Shoreline has no public write API, so the site won't integrate with it. |
| Family tree | Named families, each with big/little lineages | Matches how HKSA families work. |

## 2. Tech stack

- **UI:** Tailwind 4, shadcn/ui components, `yet-another-react-lightbox` for the gallery, `@xyflow/react` (React Flow) for the family tree
- **Data:** Supabase Postgres via Drizzle (server-side only), Row Level Security on for all tables
- **Files:** a Supabase Storage bucket `photos`. The browser uploads directly using a signed upload URL, and `next/image` renders images with the Supabase host in `remotePatterns`.
- **Auth:** Supabase Auth (Google login or magic link) via `@supabase/ssr`. Only emails in the `admins` table can access `/admin`.
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
              │
              ├──> Supabase Postgres (via Drizzle)
              ├──> Supabase Storage  (photos bucket)
              ├──> Supabase Auth     (admin login)
              └──> Behold.so / Instagram API (cached ~1h)
```

## 4. Data model (Postgres)

- `albums` (id, slug, title, title_zh, event_date, cover_photo_id)
- `photos` (id, album_id, storage_path, width, height, caption, caption_zh, taken_at, sort_order)
- `families` (id, name, name_zh, color, founded_year)
- `members` (id, name, chinese_name, grad_year, family_id, photo_path, is_visible)
- `bigs_littles` (big_id, little_id, year): lineage edges. Each little has at most one big (unique on `little_id`).
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
- `rsvps` (event_id, name, email, created_at), unique on (event_id, email)
- `officers` (id, name, role, role_zh, bio, photo_path, term such as `"2026-27"`, sort_order)
- `admins` (email)
- `instagram_cache` (id, payload jsonb, fetched_at), used once the official API is in place

## 5. Routes

**Public pages** (`app/[lang]/`):
- `/`: hero, next upcoming event, Instagram strip
- `/gallery`, `/gallery/[album]`: albums and a photo grid with a lightbox
- `/family-tree`: React Flow tree, filterable by family
- `/events`, `/events/[slug]`: event list and details, RSVP form, "add to calendar" `.ics` file
- `/board`: current officers, plus an archive of past boards
- `/suggest`: suggestion form. `?type=event|collab|feedback` preselects the category, so other pages (such as events) can link straight to it.
- `/join`: built by Kelvin (not part of this plan)

**Admin pages** (`app/admin/`): `proxy.ts` guards them, and every Server Action also calls `isAdmin()`.
- Dashboard
- Albums & photos: upload, reorder, caption
- Families, members and the big/little lineage editor
- Events and RSVP lists
- Officers
- Suggestions inbox: filter by category and status, change status, add notes, CSV export

**Route handlers:**
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

## 7. Instagram design

`lib/instagram/` exports `getLatestPosts(n)`, backed by an `InstagramProvider` interface:
- `BeholdProvider` (now): fetches a Behold.so JSON feed URL (`BEHOLD_FEED_URL`).
- `GraphProvider` (later): the official Instagram API with Instagram Login. It needs the account switched to **Business/Creator** and a long-lived token (60 days) refreshed by a Vercel Cron job.

An environment variable (`INSTAGRAM_PROVIDER`) chooses the provider. The result is cached for about an hour with Next's caching/revalidation. If the feed fails, the page falls back to a "Follow us on Instagram" link.

## 8. Phased roadmap

1. **Foundation**: Supabase project, Drizzle schema and migrations, `.env` setup, `[lang]` layout, nav/footer, brand theme
2. **Admin auth**: Supabase Auth, `admins` allowlist, `proxy.ts` guard, admin shell
3. **Gallery**: admin upload to Storage, album pages, lightbox
4. **Suggestion form**: `/suggest` with category-specific fields, Zod + Turnstile, saves to `suggestions`, admin inbox + CSV export
5. **Family tree**: families/members/lineage admin CRUD (create, read, update, delete), React Flow public view
6. **Events + RSVP**: list and detail pages, RSVP, `.ics` download
7. **Officer board**: current board plus archive
8. **Instagram**: Behold feed first, then the Graph API and token-refresh cron
9. **Polish**: 繁中 translations, SEO metadata and OG images, accessibility pass, Vercel Analytics

## 9. Environment variables (planned)

```
DATABASE_URL=                     # Supabase Postgres (pooled)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=        # server only
INSTAGRAM_PROVIDER=behold         # behold | graph
BEHOLD_FEED_URL=
INSTAGRAM_ACCESS_TOKEN=           # later, graph provider
TURNSTILE_SITE_KEY= / TURNSTILE_SECRET_KEY=
CRON_SECRET=
```

## 10. Future feature ideas

- Public "You asked, we did" board showing suggestions marked `done`
- Resend email newsletter
- Merch / dues page (Stripe payment links)
- Alumni directory (opt-in)
- Cantonese word of the week / culture blog
- Sponsor and partner org showcase
- Event check-in via QR code for attendance points
