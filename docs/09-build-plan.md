# 09 — Build Plan (execute in this exact order)

Each milestone ends with a **check** — run it before moving on. Commit after every
milestone (`git init` at M0) so a failed step never destroys prior work. Target:
all 13 milestones in one session, ending with a **live production deployment** (M12).

## M0 — Scaffold & infrastructure

- `git init` in `listening-party/app/`; create Next.js 15 app (TS, Tailwind, App Router,
  `src/` dir, no ESLint prompts blocking — accept defaults non-interactively).
- Add shadcn/ui (init + add: button, input, textarea, card, dialog, sheet, tabs,
  dropdown-menu, avatar, badge, select, sonner, skeleton, table, switch).
- Add deps: `prisma @prisma/client next-auth@beta @auth/prisma-adapter zod resend
  @aws-sdk/client-s3 @aws-sdk/s3-request-presigner jose nanoid @dnd-kit/core
  @dnd-kit/sortable qrcode recharts @tanstack/react-query date-fns` and dev:
  `vitest tsx @types/qrcode`.
- `docker-compose.yml` (01) up; `.env` from `.env.example` with generated secrets.
- Prisma schema from 02 verbatim, plus `directUrl = env("DIRECT_URL")` on the
  datasource (deployment doc 11 needs it; locally `DIRECT_URL` = `DATABASE_URL`)
  → `prisma migrate dev --name init`.
- `lib/db.ts`, `lib/storage.ts`, `lib/email.ts` (console fallback), `lib/analytics.ts`.
- Apply the **visual system from docs/12** at scaffold time: brand + room CSS-variable
  sets, Inter + Fraunces + Caveat via `next/font/google`, the 6 mood-preset `data-preset` styles,
  and the frosted-panel component. Drop the LP logo SVGs into `public/brand/`. Every
  later screen is built against these tokens — no ad-hoc colors.
- **Check:** `pnpm build` passes; `prisma studio` shows all tables; MinIO console
  reachable; a throwaway page renders the flame gradient, Inter, Fraunces, and Caveat correctly.

## M1 — Auth

Auth.js config per 06 (magic link w/ console fallback, Google conditional, Apple
conditional), Prisma adapter, session callback w/ role+handle, middleware per 01,
`requireX` helpers, signin/verify pages, unsubscribe endpoint.
**Check:** sign in via console-printed magic link end-to-end; session shows in
`/api/me`; `/studio` redirects anonymous → signin.

## M2 — Artist onboarding + Party CRUD (Module 2)

Become-an-artist flow, studio shell + dashboard, party create/edit(Details tab)/
duplicate/archive/delete, slug logic, image uploads (presign + `/api/media`),
**room theme editor** (mood-preset picker + accent swatches + background upload + intro,
persists to `Party.theme` — docs/12 screen 12), status
transitions publish/go-live/release (notifications stubbed as console logs until M8),
pre-save links editor.
**Check:** create party, upload cover, publish it; duplicate; archive; delete a draft;
verify `/join/...` 404s for drafts and renders for published.

## M3 — Experience Builder (Module 3)

`types/blocks.ts` zod union (05), block CRUD APIs, builder UI: sortable list,
add-block sheet, per-type editor forms, autosave, reorder, delete, publish validation.
**Check:** build a party containing all 15 block types, reorder by drag, rename an
act via `actLabel`, reload — order + label persist; invalid config blocks publish
with per-block errors.

## M4 — Secure audio (Module 4)

Tracks manager, audio presign w/ `tracks/` prefix, track CRUD, status/play-token/
stream/heartbeat routes per 07 (transactional play counting, token grace window,
Range support), owner-preview signed-URL path.
**Check (vitest + manual):** unit-test the play-limit transaction (parallel token
requests at limit → exactly `playLimit` Play rows) and token expiry math; manually
stream the seed MP3, confirm 429 after limit, confirm `/api/media/tracks/...` → 403.

## M5 — Join flow + the Room (Modules 1+3 fan side)

Join page (all states per 04 §3), auto-join-on-callback (06), experience payload
endpoint, the **three-pane room** per 04 §4 + 12: left act nav (done/current/locked
states, guided-first-pass then free revisit), center stage with all 15 block/act
renderers + "What happens next" storyboard, persistent bottom **snippet** player,
progress persistence (localStorage + `POST .../progress` → `Membership.furthestAct`),
finale/Stay-Connected state, complete API, poll vote + question answer APIs. Apply the
party's `theme` (mood preset/accent/background). Chat rail renders a static stub here;
it goes live in M6. Mobile: single scroll, circular act rail, inline chat, fixed
top bar + bottom snippet bar (docs/04 §4 Mobile).
**Check:** as a fan (magic link from console), join via ref link, walk every act in
order, revisit an earlier act from the sidebar, confirm a not-yet-reached act is locked,
snippet plays in the bottom bar while the secure song stays in Listen, vote, answer, hit
play limit, reach finale, `completedExperience` flips in DB; room renders in the party's
mood preset on desktop and mobile. **Listen act focus mode:** entering it dims the room
and recedes chat to a swipe-up strip; pressing play on the secure song auto-pauses the
snippet (never two audios at once); swipe-up restores chat.

## M6 — Community + Room Chat (Module 5)

Feed page, composer w/ image upload, posts/comments/likes/pin/report APIs and UI,
post search, soft delete. **Room Chat goes live:** `mode=chat` + `after=` polling on
the posts endpoint (3–5 s interval, pause when tab hidden), wire the desktop chat rail
+ mobile inline chat + CHAT act spotlight to it, presence counts (`lastSeenAt`
touch + 5-min window), chat recede/restore in the Listen act (07).
**Check:** fan posts w/ image; artist posts (badge + pin); reply thread; likes toggle;
report lands in DB; search filters; two browsers in the same room see each other's
chat messages within ~5 s and the online count reflects both.

## M7 — Pre-save, sharing, referrals (Modules 6+9)

Presave-click flow + PRESAVE_CTA block wiring, QR endpoint, share sheet on finale
(copy/native/QR + referral URL), referral attribution on join, audience page
(share card, invites, members table + CSV, leaderboard w/ badges).
**Check:** join with `?ref=` → `referredByCode` recorded and leaderboard counts it;
QR PNG downloads and encodes the join URL; invite emails print to console.

## M8 — Notifications (Module 7)

`notify.ts` service per 08, wire all five triggers, bell dropdown + `/me`
notifications, mark-read, email templates w/ unsubscribe footer, opt-out checks.
**Check:** publish/go-live/release/artist-post each produce Notification rows for the
right recipients + console emails; reply notifies parent author only; unread badge
counts and clears.

## M9 — Analytics (Module 8)

Event ingestion (client allowlist + server emits), player/join instrumentation,
analytics service queries, studio analytics page with all widgets + empty states.
**Check:** walk the fan journey again, then load analytics — funnel, joins chart,
listening stats, countries(null-safe locally), devices, referrals all render; seed
events make charts non-empty.

## M10 — Discovery, profiles, fan home, admin (Modules 10+11+12)

Landing, discover (search/trending/featured), artist profile + follow, `/me`, admin
page (overview/users/reports/flags), ban enforcement, featured flag convention.
**Check:** search finds seed artist by name and genre; follow → NEW_PARTY notification
on next publish; admin bans a user → their session dies; flag `featured:{id}` surfaces
party on discover.

## M11 — Seed, polish, full acceptance pass

Complete `prisma/seed.ts` per 02, `not-found`/`error`/`loading` states, mobile pass on
player + builder, meta/OG tags on join + artist pages (party cover as og:image), PWA
manifest + icons (no service worker), a11y sweep (focus, labels, contrast).
**Check:** run every line of `docs/10-acceptance-checklist.md` (except §Deployment)
against `pnpm dev` with a fresh `prisma migrate reset && prisma db seed`. Fix until green.

## M12 — Production deployment

Execute `docs/11-deployment.md` end to end: GitHub repo, Neon, R2 (+CORS), Resend,
Vercel env + deploy, `prisma migrate deploy`, production seed (admin =
ekowandy627@gmail.com, no demo data), streaming-route Vercel settings.
This milestone needs the user's account logins (Vercel/Neon/Cloudflare/Resend) —
pause and ask for them here, not earlier.
**Final check:** §Deployment of the acceptance checklist — real magic-link email via
Resend, full artist+fan smoke test on the live URL.

## Standing rules during the build

- Never expose track file keys; grep for `fileKey` in client components at M11.
- Every API input zod-validated; every mutation ownership-checked via `requireX`.
- Keep `lib/services/*` free of Next.js imports (pure logic over Prisma) — testable.
- If a dependency fights the one-take (e.g. Apple provider, ffmpeg), apply its
  documented fallback and keep moving; note it in a `BUILD-NOTES.md`.
