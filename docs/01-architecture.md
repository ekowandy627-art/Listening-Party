# 01 — Architecture & Project Layout

## System shape

```
Browser (fan / artist / admin)
   │  HTTPS
   ▼
Next.js app (Vercel)
   ├── React Server Components + client components (UI)
   ├── app/api/** route handlers (REST, zod-validated)
   ├── lib/services/** (business logic — pure functions over Prisma)
   ├── Auth.js (magic link + Google [+ Apple])
   │
   ├──► PostgreSQL (Neon / local Docker)         all app data
   ├──► Cloudflare R2 (S3 API / local MinIO)     media files
   └──► Resend                                    magic links + notification emails
```

There is no separate backend service, no queue, no cron worker in MVP. Everything
that "sends" (emails, notifications) happens inline in the request that triggers it,
fanned out with `Promise.allSettled` and capped at 100 recipients per batch loop.

## Directory layout (create exactly this)

```
app/                        # Next.js app  (lives at listening-party/app/)
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── docker-compose.yml      # postgres:16 + minio
├── src/
│   ├── app/
│   │   ├── (marketing)/page.tsx            # landing
│   │   ├── discover/page.tsx               # search + trending
│   │   ├── join/[handle]/[slug]/page.tsx   # public join page (email capture)
│   │   ├── p/[handle]/[slug]/              # THE ROOM (auth required; docs/04 §4)
│   │   │   ├── page.tsx                    #   three-pane room / mobile scroll (acts + chat + snippet bar)
│   │   │   └── community/page.tsx          #   community feed (same Post stream as room chat)
│   │   ├── a/[handle]/page.tsx             # public artist profile
│   │   ├── me/page.tsx                     # fan home: joined parties, notifications
│   │   ├── studio/                         # artist area (role ARTIST)
│   │   │   ├── page.tsx                    #   dashboard (party list)
│   │   │   ├── profile/page.tsx
│   │   │   └── parties/
│   │   │       ├── new/page.tsx
│   │   │       └── [partyId]/
│   │   │           ├── edit/page.tsx       #   details + experience builder
│   │   │           ├── audience/page.tsx   #   members, invites, share/QR
│   │   │           └── analytics/page.tsx
│   │   ├── admin/                          # role ADMIN
│   │   │   └── page.tsx                    #   tabbed: users/parties/reports/flags
│   │   ├── auth/
│   │   │   ├── signin/page.tsx
│   │   │   └── verify/page.tsx             #   "check your email"
│   │   └── api/                            # see 03-api-spec.md for every route
│   ├── components/                         # ui/ (shadcn), blocks/, player/, community/, studio/
│   ├── lib/
│   │   ├── auth.ts                         # Auth.js config
│   │   ├── db.ts                           # Prisma singleton
│   │   ├── storage.ts                      # R2 signed-url + proxy helpers
│   │   ├── email.ts                        # Resend wrapper w/ console fallback
│   │   ├── analytics.ts                    # trackEvent()
│   │   └── services/                       # party.ts, blocks.ts, audio.ts, community.ts,
│   │                                       # presave.ts, notify.ts, referral.ts, search.ts
│   └── types/blocks.ts                     # zod discriminated union (05-blocks-spec.md)
└── package.json
```

## Environment variables (`.env.example` — create with these exact keys)

```bash
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/listeningparty
AUTH_SECRET=                # openssl rand -base64 32
AUTH_URL=http://localhost:3000
AUTH_GOOGLE_ID=
AUTH_GOOGLE_SECRET=
AUTH_APPLE_ID=              # optional — provider disabled when empty
AUTH_APPLE_SECRET=
RESEND_API_KEY=             # optional in dev — email.ts logs to console when empty
EMAIL_FROM=parties@listeningparty.app
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY_ID=minioadmin
S3_SECRET_ACCESS_KEY=minioadmin
S3_BUCKET=listening-party
S3_PUBLIC_ENDPOINT=http://localhost:9000   # used in signed GET urls
NEXT_PUBLIC_APP_URL=http://localhost:3000
PLAY_TOKEN_SECRET=          # openssl rand -base64 32 (audio play tokens, 07-secure-audio)
```

**Dev-mode fallbacks are mandatory:** the app must run end-to-end with only
`DATABASE_URL`, `AUTH_SECRET`, `PLAY_TOKEN_SECRET`, and the MinIO defaults set.
Magic-link emails print their URL to the server console; Google/Apple buttons hide
themselves when their env vars are empty.

## docker-compose.yml

```yaml
services:
  db:
    image: postgres:16-alpine
    environment: { POSTGRES_PASSWORD: postgres, POSTGRES_DB: listeningparty }
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment: { MINIO_ROOT_USER: minioadmin, MINIO_ROOT_PASSWORD: minioadmin }
    ports: ["9000:9000", "9001:9001"]
    volumes: [miniodata:/data]
volumes: { pgdata: {}, miniodata: {} }
```

Seed script must create the `listening-party` bucket in MinIO if missing
(`CreateBucketCommand`, ignore "already exists").

## Middleware & access control

`src/middleware.ts` protects route groups:

| Path prefix | Requirement | On failure |
|---|---|---|
| `/me`, `/p/**` | signed in | redirect to `/join/[handle]/[slug]` if inside a party path (preserves invite UX), else `/auth/signin?callbackUrl=…` |
| `/studio/**` | role `ARTIST` or `ADMIN` | signed-in fans see a "Become an artist" page at `/studio` (creates ArtistProfile); anonymous → signin |
| `/admin/**` | role `ADMIN` | 404 (don't reveal it exists) |
| `/api/**` | per-route (03-api-spec.md) | 401/403 JSON |

Ownership checks happen in services, not middleware: every studio mutation verifies
`party.artistProfile.userId === session.user.id` (admins bypass).

## Performance & non-functional targets

- All public pages (`/`, `/discover`, `/a/*`, `/join/*`) are server-rendered and cacheable; target LCP < 2 s on Fast 3G emulation.
- Images served via `next/image` with R2 signed URLs (add R2/MinIO host to `images.remotePatterns`).
- Mobile-first: every screen designed at 390 px first; the room is a single vertical
  scroll between a fixed top bar and the bottom snippet player on mobile, three panes +
  bottom player on desktop (docs/04 §4, docs/12).
- Accessibility: all interactive elements keyboard-reachable, focus rings visible, media blocks have text alternatives, color contrast AA. shadcn/ui defaults cover most of this — don't strip its aria attributes.
- GDPR: `/api/me` DELETE anonymizes the user (email → `deleted-{cuid}@deleted`, content kept but attributed to "Deleted user"); privacy page stub at `/privacy`.
