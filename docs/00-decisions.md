# 00 — Final Tech Decisions

Every choice below is **final** for the MVP build. Do not re-open them during the
build session. Deviations from the PRD are listed with reasons at the bottom.

## Stack

| Concern | Decision | Notes |
|---|---|---|
| Framework | **Next.js 15+ (App Router, TypeScript, single full-stack app)** | API route handlers replace a separate NestJS backend for MVP |
| Styling | **Tailwind CSS v4 + shadcn/ui** | Radix-based components. Two surfaces (brand + room) and 6 room mood-presets — full visual system in **docs/12-theming.md** (branding is now LOCKED: LP logo, Inter + Caveat, coral→amber flame gradient) |
| Database | **PostgreSQL 16** | Local: Docker Compose. Prod: Neon (works with Vercel) |
| ORM | **Prisma** | Schema in `docs/02-data-model.md` is copied verbatim |
| Auth | **Auth.js (NextAuth v5)** — Email (magic link via Resend), Google, Apple providers | Database session strategy (Prisma adapter) so sessions can be revoked |
| File storage | **Cloudflare R2 via S3 API** (`@aws-sdk/client-s3`) | Local dev: MinIO in Docker Compose, same S3 API |
| Audio streaming | **Server-proxied byte-range streaming** through an authenticated API route with short-lived one-time play tokens | See `docs/07-secure-audio-spec.md`. Encrypted HLS is deferred (deviation D3) |
| Email | **Resend** | Magic links + notification emails. Dev fallback: log the email payload to console when `RESEND_API_KEY` is unset. **User confirmed a real Resend key is in scope** — production sends real email (doc 11 step 4 covers the no-custom-domain case) |
| Validation | **Zod** on every API input; block configs are discriminated unions | |
| Drag & drop | **@dnd-kit/core + @dnd-kit/sortable** | Experience Builder reordering |
| Media player | Native `<audio>`/`<video>` elements wrapped in custom components | No third-party player |
| QR codes | **`qrcode`** npm package, generated server-side | |
| Charts (analytics) | **Recharts** | |
| State | React Server Components + server actions where natural; **TanStack Query** for the few polling/interactive views (community feed, player) | No Redux |
| IDs | Prisma `cuid()` | |
| Testing | **Vitest** for unit (playback-rule engine, zod schemas, referral codes) + a manual acceptance checklist | No Playwright in MVP one-take |
| Hosting target | Vercel + Neon + R2 | **Production deploy is part of the one-take build** (milestone M12, doc 11) — the session ends with a live URL, not just local dev |

## Project location & name

Create the app at `listening-party/app/` inside this folder (sibling of `docs/`).
Package name `listening-party`. Node 20+, package manager: **pnpm** (fall back to npm
if pnpm is unavailable on the machine).

## Roles

Single `User` table with `role: FAN | ARTIST | ADMIN`. Any fan can upgrade to artist
by creating an artist profile ("Become an artist" flow). Admin is set manually in DB
(seed creates one).

## Conventions

- All API routes under `app/api/**` return `{ data }` on success, `{ error: { code, message } }` on failure, with proper HTTP status.
- All timestamps stored UTC, rendered in the viewer's locale.
- Slugs/handles: lowercase, `[a-z0-9-]`, unique, auto-generated from title/name with collision suffix (`-2`, `-3`).
- Media keys in R2: `{artistId}/{partyId}/{uuid}.{ext}` — never expose raw R2 URLs to the client for audio; images/video may use signed GET URLs (1 h expiry).
- Feature flags: simple `FeatureFlag` table read server-side; no third-party service.

## Deviations from the PRD (final)

| # | PRD says | MVP builds | Why |
|---|---|---|---|
| D1 | NestJS/Node backend + Next.js frontend | Single Next.js full-stack app | One deployable, one repo, dramatically better odds of a working one-take build; API layer is still cleanly separated under `app/api` and `lib/services` so it can be extracted later |
| D2 | PWA | Responsive web only; add manifest + icons but no service worker/offline | Service workers add build complexity with near-zero MVP value |
| D3 | Encrypted HLS + watermarking + screen-recording detection | Proxied range streaming with one-time play tokens, no direct URL, download/right-click blocked in UI, play limits enforced **server-side** | HLS transcoding pipeline (ffmpeg workers) is not one-take feasible; the server-side play-token model already delivers the real security property (no URL exposure, enforced limits). HLS slots in later behind the same token API |
| D4 | Push notifications | Email only; in-app notification bell (DB-backed) covers the "instant" feel | Web push needs service worker + VAPID setup (see D2) |
| D5 | Spotify auto pre-save | Single pre-save link per platform pasted by artist; fan clicks out | PRD itself marks auto pre-save as Future |
| D6 | SMS/WhatsApp | Not built | PRD marks as Future |
| D7 | Cities-level geo analytics | Country + device + referrer only | Country comes free from Vercel/`accept-language`+IP header; city needs a geo-IP DB — cut |
| D8 | Screen recording detection | Not built | Not reliably possible on web |
| D9 | Merch module | Merch **Promotion block** only (image + text + external link) | Store is Phase 3 |
| D10 | "Users can later create passwords" | Not built — magic link + OAuth only | Password auth adds reset flows, hashing, and forms for no MVP gain; Auth.js makes adding it later trivial |
| D11 | Apple Sign-In | Wired in code but disabled unless `APPLE_ID`/`APPLE_SECRET` env vars are present | Apple credentials require a paid developer account; the build must not block on it. Google + magic link are the working paths |
