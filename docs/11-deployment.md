# 11 — Deployment (production is part of "done")

The one-take build ends with a **live production URL**, not just local dev. This doc
is the exact path. Everything here is executed as milestone **M12** in the build plan.

## Confirmed for this launch (2026-07-03)

- **No custom domain** — ships on `{project}.vercel.app`. Nothing in the code depends
  on a domain (`NEXT_PUBLIC_APP_URL`/`AUTH_URL` are env-driven), so one can be attached
  anytime later with zero rework.
- **No Google/Apple OAuth** — magic link only for launch. The buttons simply don't
  render (00-decisions D11 mechanics); adding Google later is a ~10-minute add, no
  code changes beyond setting two env vars.
- **Real email via Resend**, but since there's no domain, `EMAIL_FROM` uses Resend's
  shared `onboarding@resend.dev` sender — **this can only deliver to the Resend
  account owner's own inbox.** That's sufficient for the artist's own testing and the
  M12 smoke test, but fans with other email addresses won't receive real emails until
  a domain is bought and verified. Flagged as the first post-launch upgrade.
- **Production seed uses the real launch artist**, not the demo: artist name **"Challe
  from Ghana"**, handle **`challefromghana`** (confirm/adjust at M12 if a shorter handle
  is wanted). Song file, cover art, and story text are supplied by the user when M12 runs.

## Prerequisites — none of the four accounts exist yet

All four are free-tier signups with no cost commitment (R2 asks for a card on file but
free tier covers this project's usage). Do these **before or at the start of M12** —
they're plain account creation, no code involved, so they can happen anytime before then:

| # | Service | Sign up at | Notes |
|---|---|---|---|
| 1 | **GitHub** | github.com (skip if already have one) | Needed to push the repo Vercel deploys from |
| 2 | **Vercel** | vercel.com/signup → "Continue with GitHub" | Hosting. Connecting via GitHub avoids a separate password |
| 3 | **Neon** | neon.tech → sign up (GitHub login works here too) | Production Postgres. Free tier: 1 project, enough for MVP |
| 4 | **Cloudflare** | dash.cloudflare.com/sign-up → then enable **R2** in the dashboard | Media storage. R2 free tier: 10 GB storage + generous egress — R2 may ask for a card even on the free tier, that's normal |
| 5 | **Resend** | resend.com/signup | Email. No domain needed yet — the shared sender works out of the box |

Order matters a little: do GitHub → Vercel first (Vercel account creation is instant via
GitHub OAuth), then Neon/Cloudflare/Resend in any order. None of this requires me — it's
just clicking "sign up" on each site. Come back to M12 once all five exist.

## M12 step-by-step

1. **Repo**: push `app/` to a new GitHub repo (private), `main` branch.
   (`gh repo create listening-party --private --source . --push`)
2. **Neon**: create project `listening-party` → copy the **pooled** connection string
   (with `-pooler`, `sslmode=require`) for the app; also copy the **direct** (unpooled)
   string for migrations (`DIRECT_URL`). Add to Prisma datasource:
   `directUrl = env("DIRECT_URL")` — add this at M0 so schema needs no change later;
   locally set `DIRECT_URL=DATABASE_URL`.
3. **R2**: create bucket `listening-party` → create API token scoped to the bucket
   (Object Read & Write) → note `S3_ENDPOINT=https://{accountid}.r2.cloudflarestorage.com`.
   Apply CORS on the bucket (required for browser presigned PUT uploads):
   ```json
   [{ "AllowedOrigins": ["https://{app-domain}"],
      "AllowedMethods": ["PUT", "GET"],
      "AllowedHeaders": ["*"], "MaxAgeSeconds": 3600 }]
   ```
   (Add `http://localhost:3000` to AllowedOrigins too, so local dev can also point at
   R2 if ever needed.)
4. **Resend**: create API key. No domain → `EMAIL_FROM=onboarding@resend.dev`
   (delivers to the account owner's inbox only — confirmed acceptable for this
   launch; see note above). Buying + verifying a domain later just means changing
   this one env var and redeploying.
5. **Vercel**: `vercel link` → set env vars (Production):
   `DATABASE_URL` (pooled), `DIRECT_URL`, `AUTH_SECRET` (fresh `openssl rand -base64 32`),
   `AUTH_URL=https://{project}.vercel.app`, `NEXT_PUBLIC_APP_URL=https://{project}.vercel.app`,
   `PLAY_TOKEN_SECRET` (fresh), `RESEND_API_KEY`, `EMAIL_FROM=onboarding@resend.dev`,
   `S3_ENDPOINT`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`,
   `S3_BUCKET=listening-party`, `S3_PUBLIC_ENDPOINT={S3_ENDPOINT}`.
   Google/Apple vars stay unset — their buttons hide (00-decisions D11 mechanics).
6. **Migrate + seed prod**: from local machine with prod vars in `.env.production.local`:
   `prisma migrate deploy`, then run the **production seed** (see below) — *not* the
   demo seed.
7. **Deploy**: `vercel --prod`. Set the deployed URL back into `AUTH_URL` /
   `NEXT_PUBLIC_APP_URL` if it differs from the guess in step 5, redeploy if changed.
8. **Production smoke test** (checklist §Deployment in doc 10): magic-link sign-in
   with the user's real email, become-artist, create a tiny party, upload an image +
   audio, publish, join it from a second (incognito) session via magic link, play the
   track, post in community, check the analytics page.

## Production seed (`prisma/seed-prod.ts`)

Unlike the local demo seed, production gets:
- Admin user: `ekowandy627@gmail.com` (`role=ADMIN`) — the user signs in with a real
  magic link and inherits admin.
- **The real launch artist**: ArtistProfile `name: "Challe from Ghana"`,
  `handle: "challefromghana"` (confirm/shorten at M12 if desired), owned by the admin
  user (or a separate account if the artist signs in with a different email — ask at
  M12). One DRAFT party is scaffolded so the smoke test only has to fill in the song,
  cover art, and story text rather than build a party from zero — but the artist
  finishes and publishes it live, during the smoke test, with real assets.
- Feature flags table left empty.
No fake fans/demo party in prod — memberships come from the real smoke test.

## Streaming note for Vercel

The `/api/stream/[token]` proxy must set `export const dynamic = "force-dynamic"` and
stream the R2 body through (`new Response(body, …)` with the ReadableStream — do not
buffer the whole file). Vercel functions cap response duration; with ~50 MB max audio
and range requests this is fine on the default plan, but set
`export const maxDuration = 60` on the stream route defensively.

## Rollback / redeploy

All state is in Neon + R2; the app is stateless. Rollback = Vercel "promote previous
deployment". Migrations are forward-only in MVP (no down migrations) — acceptable at
this stage.
