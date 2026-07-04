# 03 — API Specification

All routes live under `src/app/api/`. Conventions:

- Success: `200/201` with `{ "data": … }`. Errors: `{ "error": { "code", "message" } }`
  with status 400 (validation), 401 (no session), 403 (wrong role/not owner/not member),
  404, 409 (conflict), 410 (gone/expired), 429 (play limit reached).
- Every body is validated with zod; validation failures return code `VALIDATION`.
- **Auth column**: `public` | `session` (any signed-in user) | `member` (session + Membership
  in the party) | `owner` (session + owns the party via ArtistProfile; ADMIN bypasses) |
  `admin`.
- Server actions may be used instead of fetch for studio form submissions, but they must
  call the same `lib/services/*` functions — the service layer is the contract.

## Auth (handled by Auth.js)

| Route | Notes |
|---|---|
| `GET/POST /api/auth/[...nextauth]` | Auth.js catch-all: magic link (Email provider), Google, Apple |

Magic-link sign-in from a join page must pass `callbackUrl=/p/{handle}/{slug}?ref={code?}`
so the fan lands inside the party after clicking the emailed link.

## Me / account

| Method & path | Auth | Body → Response |
|---|---|---|
| `GET /api/me` | session | → `{ user, artistProfile?, memberships: [{party, joinedAt}], unreadNotifications: n }` |
| `PATCH /api/me` | session | `{ name?, image? }` → updated user |
| `DELETE /api/me` | session | GDPR anonymize (01-architecture) → `{ ok: true }` |
| `GET /api/me/notifications?cursor=` | session | → `{ items: Notification[], nextCursor }` (20/page) |
| `POST /api/me/notifications/read` | session | `{ ids: string[] } | { all: true }` → `{ ok }` |

## Artist profile

| Method & path | Auth | Body → Response |
|---|---|---|
| `POST /api/artists` | session | `{ name, handle?, bio?, genres? }` → creates ArtistProfile, sets `role=ARTIST` (409 if handle taken; auto-suffix when handle omitted) |
| `PATCH /api/artists/me` | owner-of-profile | `{ name?, bio?, genres?, links?, photo? }` → profile |
| `GET /api/artists/[handle]` | public | → `{ profile, upcoming: Party[], past: Party[], followerCount, isFollowing }` (only non-draft, non-archived parties) |
| `POST /api/artists/[handle]/follow` | session | toggle → `{ following: boolean }` |

## Parties (studio)

| Method & path | Auth | Body → Response |
|---|---|---|
| `GET /api/parties` | owner | list own parties incl. drafts/archived → `{ items }` |
| `POST /api/parties` | ARTIST | `{ title, description?, genre?, releaseDate? }` → party (slug auto from title) |
| `GET /api/parties/[id]` | owner | full party incl. blocks (ordered), tracks (without fileKey), preSaveLinks |
| `PATCH /api/parties/[id]` | owner | any dashboard field incl. `slug`, `theme` (mood preset/accent/background/intro — docs/12), `snippetKey` (teaser for the bottom player — presigned as `kind:"audio"`, stored plain, NOT play-limited) → party |
| `POST /api/parties/[id]/duplicate` | owner | → new DRAFT party, copies blocks/tracks/pre-save links, title "Copy of …" |
| `DELETE /api/parties/[id]` | owner | hard delete, only when `status=DRAFT`; else 409 telling artist to archive |
| `POST /api/parties/[id]/archive` | owner | sets/unsets `archivedAt` (body `{ archived: boolean }`) |
| `POST /api/parties/[id]/publish` | owner | DRAFT→PUBLISHED. Validates: ≥1 block, title, coverArt. Sends `NEW_PARTY` notification to artist's followers |
| `POST /api/parties/[id]/go-live` | owner | PUBLISHED→LIVE. Sends `PARTY_LIVE` to members |
| `POST /api/parties/[id]/release` | owner | LIVE or PUBLISHED→COMPLETED. Sends `SONG_RELEASED` to members (email includes pre-save/streaming links) |

## Blocks

| Method & path | Auth | Body → Response |
|---|---|---|
| `POST /api/parties/[id]/blocks` | owner | `{ type, config }` (zod per type) → block appended at end |
| `PATCH /api/blocks/[blockId]` | owner | `{ config }` → block |
| `DELETE /api/blocks/[blockId]` | owner | → `{ ok }` |
| `POST /api/parties/[id]/blocks/reorder` | owner | `{ orderedIds: string[] }` → reindexes `order` 0..n in a transaction (400 if ids ≠ exactly the party's block ids) |
| `GET /api/parties/[id]/experience` | member (owner bypass = preview mode) | full room payload: party header + `theme` + snippet signed URL (if `snippetKey`), ordered acts/blocks w/ media keys resolved to signed URLs, per-user poll/question state + `furthestAct`, track statuses, artist header (05), **presence** (`{ online, memberCount, countryCount }` — online = memberships w/ `lastSeenAt` in last 5 min; countryCount = distinct non-null countries on the party's `party_joined` events). Fetching this touches the caller's `lastSeenAt`. One round-trip renders the room |
| `POST /api/parties/[id]/progress` | member | `{ actIndex }` → monotonic max on `Membership.furthestAct` (clamped to block count) → `{ furthestAct }` |
| `POST /api/blocks/[blockId]/vote` | member | `{ optionIdx }` → upserts PollVote → `{ counts: number[] , myVote }` |
| `POST /api/blocks/[blockId]/answer` | member | `{ answer (1..2000) }` → upserts QuestionAnswer → `{ ok }` |

## Media upload

Two-step: presign → client PUTs directly to R2/MinIO → confirm key on the entity.

| Method & path | Auth | Body → Response |
|---|---|---|
| `POST /api/uploads/presign` | session | `{ kind: "image"\|"video"\|"audio"\|"snippet", contentType, sizeBytes, partyId? }` → `{ uploadUrl, key }`. Enforce: image ≤ 10 MB, audio/snippet ≤ 50 MB, video ≤ 200 MB; contentType allowlist (`image/jpeg,png,webp`, `audio/mpeg,mp4,wav`, `video/mp4,webm`). `kind!=image` requires party ownership. **Key prefixes:** `audio` → `tracks/…` (secure, never plain-served); `snippet` → `snippets/…` (teaser, plain-served via `/api/media` like images/video) |
| `GET /api/media/[key…]` | session | 302 → signed GET url (1 h). Used for images/videos. **Never valid for track fileKeys** (keys under `tracks/` prefix are rejected 403 — audio only flows through play tokens, see 07) |

## Tracks & playback (details in 07)

| Method & path | Auth | Body → Response |
|---|---|---|
| `POST /api/parties/[id]/tracks` | owner | `{ title, fileKey, mimeType, durationSec?, playRule, playLimit?, availableFrom?, expiresAt? }` → track (fileKey must be a key this artist presigned) |
| `PATCH /api/tracks/[trackId]` | owner | rule fields → track |
| `DELETE /api/tracks/[trackId]` | owner | → `{ ok }` |
| `GET /api/tracks/[trackId]/status` | member | → `{ playsUsed, playsAllowed: number\|null, canPlay: boolean, reason? }` |
| `POST /api/tracks/[trackId]/play-token` | member | `{ fingerprint? }` → `{ token, expiresIn: 30 }` or 429/410 with reason. Creates the `Play` row |
| `GET /api/stream/[token]` | token itself | Range-supporting audio proxy (07). No session cookie needed — token is the auth |
| `POST /api/plays/[playId]/heartbeat` | member (own play) | `{ completedPct }` → `{ ok }` (monotonic max) |

## Join & membership

| Method & path | Auth | Body → Response |
|---|---|---|
| `GET /api/join/[handle]/[slug]` | public | party join-card data: `{ title, artistName, coverArt (signed), description, releaseDate, memberCount, countryCount, recentAvatars (≤6 signed), status, theme }`. 404 if DRAFT/archived |
| `POST /api/join/[handle]/[slug]` | session | `{ ref? }` → creates Membership (idempotent), stores `referredByCode`, emits `party_joined` event → `{ membership }` |
| `POST /api/parties/[id]/invites` | owner | `{ emails: string[] }` (≤100) → sends invite emails w/ join link, emits `invite_sent` events → `{ sent: n }` |
| `GET /api/parties/[id]/qr` | public (published) | PNG QR of the join URL (`Content-Type: image/png`) |
| `POST /api/parties/[id]/presave-click` | member | `{ platform }` → sets `Membership.preSavedAt` (first click), emits `presave_click` → `{ ok }` |
| `POST /api/parties/[id]/complete` | member | marks `completedExperience=true`, emits `experience_completed` → `{ ok }` |

## Community

| Method & path | Auth | Body → Response |
|---|---|---|
| `GET /api/parties/[id]/posts?cursor=&q=&mode=` | member | **One endpoint, two renderings** (Room Chat and Community are the same Post stream — 02). Default `mode=feed`: pinned-first, newest-first, 20/page, `q` = ILIKE search on body. `mode=chat`: chronological ascending, no pinned-first reorder (pinned returned via a separate `pinned` field), supports `after={postId}` for the 3–5 s **polling** loop (returns only newer posts, ≤50). Polling with `mode=chat` also touches the caller's `lastSeenAt` and returns `presence.online`. Each item: author (name/image/isArtist), body, mediaUrl (signed), likeCount, likedByMe, commentCount → `{ items, nextCursor?, pinned?, presence? }` |
| `POST /api/parties/[id]/posts` | member | `{ body (1..2000), mediaKey? }` → post. If author is the party's artist → notify members (`ARTIST_POSTED`) |
| `DELETE /api/posts/[postId]` | author, owner, or admin | soft delete |
| `POST /api/posts/[postId]/pin` | owner | `{ pinned: boolean }` → post |
| `GET /api/posts/[postId]/comments` | member | full tree (one nesting level) |
| `POST /api/posts/[postId]/comments` | member | `{ body (1..1000), parentId? }` → comment. Notifies post author / parent-comment author (`COMMUNITY_REPLY`, never self) |
| `POST /api/posts/[postId]/like`, `POST /api/comments/[id]/like` | member | toggle → `{ liked, count }` |
| `POST /api/posts/[postId]/report` | member | `{ reason (1..500) }` → `{ ok }` |

## Pre-save links (studio)

| Method & path | Auth | Body → Response |
|---|---|---|
| `PUT /api/parties/[id]/presave-links` | owner | `{ links: [{platform, url}] }` full replace; url must be https and host-validated per platform (spotify.com/open.spotify.com, music.apple.com, amazon.\*, music.youtube.com, deezer.com) → `{ links }` |

## Search & discovery

| Method & path | Auth | Body → Response |
|---|---|---|
| `GET /api/search?q=&type=artist\|party\|all` | public | ILIKE across artist name/handle/genres and party title/genre (non-draft) → `{ artists, parties }` |
| `GET /api/discover` | public | `{ trending: Party[] (most joins last 7d), featured: Party[] (FeatureFlag-driven list via flag `featured:{partyId}`), newest: Party[] }` |

## Analytics

| Method & path | Auth | Body → Response |
|---|---|---|
| `POST /api/events` | public | `{ type, partyId?, meta? }` — allowlisted client event types only (08) → `{ ok }` (fire-and-forget; country/device/referrer derived server-side) |
| `GET /api/parties/[id]/analytics` | owner | full dashboard payload (08 defines every metric + shape) |

## Admin

| Method & path | Auth | Body → Response |
|---|---|---|
| `GET /api/admin/overview` | admin | totals: users, artists, parties by status, posts, plays (last 30 d sparkline) |
| `GET /api/admin/users?q=&cursor=` | admin | list; `PATCH /api/admin/users/[id]` `{ role?, banned? }` — banning deletes the user's sessions and blocks future signin via the Auth.js `signIn` callback (`User.banned` field, defined in 02) |
| `GET /api/admin/reports?resolved=` | admin | reports w/ post preview; `POST /api/admin/reports/[id]/resolve` `{ deletePost: boolean }` |
| `GET/PUT /api/admin/flags` | admin | list / upsert FeatureFlags |
