@AGENTS.md

# HKSA Website

Website for the Hong Kong Student Association at UCSB. It has a photo gallery, an Instagram feed, a family tree editable by admins, a suggestion form (feedback / event ideas / collab ideas), events with RSVP, an officer board, and English/繁中 support. See `plan.md` for the full architecture and roadmap.

## Stack
- Next.js 16 App Router, React 19, TypeScript, Tailwind 4, shadcn/ui
- Supabase: Postgres (via Drizzle ORM), Storage (`photos` bucket), Auth (admin login)
- Hosted on Vercel; scheduled jobs use Vercel Cron
- Zod for validation, React Flow (`@xyflow/react`) for the family tree, Cloudflare Turnstile for spam

## Layout (planned)
- `app/[lang]/`: public pages (`en`, `zh-Hant`)
- `app/admin/`: admin UI
- `app/api/`: cron and CSV export route handlers
- `lib/db/`: Drizzle schema, client, queries
- `lib/supabase/`: server/browser Supabase clients, `isAdmin()`
- `lib/instagram/`: `InstagramProvider` interface + Behold/Graph implementations
- `dictionaries/`: `en.json`, `zh-Hant.json`

## Rules
- Read the relevant guide in `node_modules/next/dist/docs/` before using a Next.js API. Middleware is now **`proxy.ts`**.
- DB access and the Supabase service-role key are **server-only**. Never import them into Client Components.
- Every admin Server Action and route handler must call `isAdmin()` itself. Don't rely on `proxy.ts` alone.
- Validate all form input and Server Action args with Zod.
- Image files go in Supabase Storage. Postgres stores only paths and metadata.
- Every user-facing string goes in both `en` and `zh-Hant` dictionaries. DB content uses optional `_zh` columns.
- Keep Row Level Security enabled on all tables.

## Commands
- `npm run dev` / `npm run build` / `npm run lint`
- Drizzle (once added): `npm run db:generate`, `npm run db:migrate`

## Known limitations
- **`/join` page:** Kelvin is building it by hand. Don't create or modify it unless asked. There's no Shoreline API integration.
- **Suggestion form (`/suggest`):** one form with `category` = `feedback` | `event` | `collab`, validated with a Zod discriminated union. Name and email are optional (anonymous allowed).
- **Instagram:** currently uses a Behold.so feed. Switch to the official API (`INSTAGRAM_PROVIDER=graph`) once the account is Business/Creator.
