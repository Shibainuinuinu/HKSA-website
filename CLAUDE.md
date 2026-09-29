@AGENTS.md

# HKSA Website

Website for the Hong Kong Student Association at UCSB. It has a photo gallery, an Instagram feed, a family tree editable by admins, a suggestion form (feedback / event ideas / collab ideas), events with RSVP, an officer board, and English/繁中 support. See `plan.md` for the full architecture, data model, roadmap and open items.

## Workflow
- Kelvin wants design and architecture agreed **before** implementation. When asked to discuss or plan, produce a plan only and don't write code.
- Build the roadmap phases in `plan.md` §9 in order. There's no rush to launch.

## Stack
- Next.js 16 App Router, React 19, TypeScript, Tailwind 4, shadcn/ui
- Supabase: Postgres (via Drizzle ORM), Storage (`photos` bucket), Auth (admin login via Google)
- Hosted on Vercel; scheduled jobs use Vercel Cron
- Zod for validation, React Flow (`@xyflow/react`) for the family tree, Cloudflare Turnstile for spam

## Layout (planned)
- `app/[lang]/`: public pages (`en`, `zh-Hant`)
- `app/admin/`: admin UI
- `app/api/`: cron (`keepalive`, `instagram-refresh`) and CSV export route handlers
- `components/`: presentational components only. They take props in and never import `lib/db` or anything server-only.
- `lib/db/`: Drizzle schema, client, queries
- `lib/supabase/`: server/browser Supabase clients, `isAdmin()`
- `lib/instagram/`: `InstagramProvider` interface + Behold/Graph implementations
- `dictionaries/`: `en.json`, `zh-Hant.json`

## Rules
- Read the relevant guide in `node_modules/next/dist/docs/` before using a Next.js API. Middleware is now **`proxy.ts`**.
- DB access and the Supabase service-role key are **server-only**. Never import them into Client Components.
- **Authorization is `isAdmin()`.** Drizzle connects as the postgres role and bypasses RLS, so every admin Server Action and route handler must call `isAdmin()` itself. Don't rely on `proxy.ts` alone.
- Keep Row Level Security enabled on all tables, with **no anon policies**. Tables must be unreachable with the public anon key.
- Validate all form input and Server Action args with Zod.
- Every user-facing string goes in both `en` and `zh-Hant` dictionaries. DB content uses optional `_zh` columns.

### Styling (deferred)
Kelvin and the publicity team are designing the UI/UX. **Don't invent a visual design.**
- Use semantic, accessible HTML. Use Tailwind for layout only (flex/grid, spacing, breakpoints), mobile-first.
- Colors, fonts and radii come only from `@theme` tokens in `app/globals.css`. Never hardcode colors.
- Pages and Server Actions fetch and mutate data, and `components/` renders it. A redesign should only touch `components/` and `globals.css`.
- Use shadcn/ui primitives for form controls, dialogs and selects.

### Images
- Image files go in Supabase Storage. Postgres stores only paths and metadata (`storage_path`, `thumb_path`).
- The browser resizes each photo at upload time (thumbnail + display size, converting HEIC → JPEG/WebP), then uploads with a signed upload URL.
- Serve gallery images straight from Storage (`unoptimized` or a custom loader), **not** through Vercel image optimization. The Hobby quota is limited.

### Family tree invariants (enforce in admin Server Actions)
- One big per little: `bigs_littles.little_id` is unique.
- A little inherits the big's family. `members.family_id` is set only on founders (members with no big) and is null for everyone else. Work out a member's family by walking up to the root.
- Reject any edge that would create a cycle. Deleting or re-parenting a member must leave the tree valid.
- Members are **visible by default** (`is_visible = true`). The admin UI must offer a quick hide toggle.

## Commands
- `npm run dev` / `npm run build` / `npm run lint`
- Drizzle (once added): `npm run db:generate`, `npm run db:migrate`

## Known limitations
- **`/join` page:** Kelvin is building it by hand. Don't create or modify it unless asked. There's no Shoreline API integration.
- **Suggestion form (`/suggest`):** one form with `category` = `feedback` | `event` | `collab`, validated with a Zod discriminated union. Name and email are optional (anonymous allowed).
- **Instagram:** currently uses a Behold.so feed. Switch to the official API (`INSTAGRAM_PROVIDER=graph`) once the account is Business/Creator.
- **Supabase free tier** pauses after about 7 days without activity. The daily `/api/cron/keepalive` job (protected by `CRON_SECRET`) prevents this.
