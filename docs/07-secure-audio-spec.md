# 07 — Secure Audio Spec

Goal (PRD Module 4): fans can stream unreleased tracks, with **no URL exposure, no
download path, and server-enforced play limits/expiry**. MVP mechanism: proxied
byte-range streaming gated by short-lived one-time play tokens (decision D3 — HLS
comes later behind the same token API).

## Two kinds of audio — do not confuse them

| | **Snippet** (`Party.snippetKey`) | **Secure track** (`Track`) |
|---|---|---|
| Purpose | teaser for the persistent bottom player; "music is always on" vibe | the unreleased song, heard in the **Listen** act |
| Security | **none** — freely playable; presigned with `kind:"snippet"` → key prefix `snippets/…`, served via `/api/media` like images/video (never under `tracks/`) | full play-token pipeline below (no URL, play limits, expiry); presigned with `kind:"audio"` → `tracks/…` prefix |
| Length | short hook (artist uploads a clipped teaser) | full song |
| Where | bottom player bar, every act | AUDIO_PLAYER block in the Listen act only |

Everything in the rest of this doc is about the **secure track**. The snippet is just a
media file served through `/api/media/[key]` — it is intentionally unprotected, because
it is a teaser the artist chose to share. Never put the secure track in the bottom player.

## Upload (artist)

1. Studio Tracks manager (inside the edit page's Experience tab or a small "Tracks"
   section on the Details tab — builder's choice, keep it one screen) →
   `POST /api/uploads/presign` with `kind: "audio"` and `partyId`.
2. Keys for audio are prefixed `tracks/` (`tracks/{artistId}/{partyId}/{uuid}.mp3`) —
   this prefix is what `/api/media/[key]` rejects, guaranteeing audio never gets a
   plain signed URL.
3. Client PUTs the file, reads duration via `<audio>` metadata locally, then
   `POST /api/parties/[id]/tracks` with the key + rule fields.

Rule fields map to PRD playback rules:
- Unlimited → `playRule: UNLIMITED`
- One/Two/Three plays → `playRule: LIMITED, playLimit: 1|2|3` (any positive int allowed)
- Date limited → `availableFrom` and/or `expiresAt`
- Time limited → same `expiresAt` mechanism (UI offers "until release date" shortcut
  that copies `party.releaseDate`)

## Playback flow (fan)

```
AUDIO_PLAYER block mounts
  → GET /api/tracks/[id]/status        { playsUsed, playsAllowed, canPlay, reason? }
  → user taps Play
  → POST /api/tracks/[id]/play-token   { fingerprint }        ─┐ atomic
       server: inside a transaction —                            │
         1. re-check membership, window (availableFrom/expiresAt)│
         2. count Plays for (track,user); if LIMITED and         │
            count >= playLimit → 429 { code:"PLAY_LIMIT" }       │
         3. create Play row                                     ─┘
         4. sign token: HMAC-SHA256(PLAY_TOKEN_SECRET) over
            { playId, trackId, userId, exp: now+30s } (JWT via `jose`)
  → <audio src="/api/stream/{token}">
  → GET /api/stream/[token]
       verify sig + exp (30 s covers only the INITIAL request;
       see "range requests" below), look up Play + Track,
       proxy R2 GetObject with Range passthrough,
       return 206/200 with Accept-Ranges, Content-Type,
       Cache-Control: private, no-store
  → every 10 s + on pause/end: POST /api/plays/[playId]/heartbeat { completedPct }
```

**Range requests after expiry:** browsers issue multiple range requests over a long
listen. The 30 s exp applies to *token freshness at first use*; on first valid use the
stream route stamps `Play.startedAt`-based grace — accept the same token for
`durationSec + 120 s` after the Play row's creation, then 410. This keeps tokens
non-shareable in practice (30 s to start, bound to one Play row, private cache) without
breaking seeks. Implement as: token valid iff `now < play.startedAt + max(duration+120, 300)s`
AND token's `exp` was valid when the Play was created (the POST already guaranteed that).

**What counts as a play:** the token POST (i.e., pressing Play). Pausing/resuming
within the same Play row does not consume another play; the client keeps the same
`<audio>` element and token for the session. Re-mounting the block (page reload) and
pressing Play again = a new play. This is simple, explainable to artists ("2 plays"
= pressing play twice), and race-safe because the count+insert is transactional
(use `SELECT count FOR UPDATE`-equivalent: wrap in `prisma.$transaction` with
`Serializable` isolation, catch retry once).

## Client-side hardening (UI layer, best-effort by design)

- `<audio>` gets `controlsList="nodownload noplaybackrate"`, `disableRemotePlayback`,
  custom controls (no native context UI), `onContextMenu` preventDefault on the whole
  player block.
- No track fileKey, R2 URL, or bucket name ever appears in any API response or HTML.
- `fingerprint`: cheap hash of `userAgent + screen + timezone` generated client-side,
  stored on the Play row for the analytics "devices" view and future abuse review. Not
  a security control; do not block on mismatch in MVP.

## The Listen act — the signature screen (approved mockup, 2026-07-03)

When the fan enters the Listen act, the room shifts into **focus mode**:

- **Eyebrow:** "◍ THE LISTEN ACT" pill; instruction line "Be the first to hear. Share
  your real reactions. Chat will return after the song."
- **Hero secure player:** large square cover art with warm glow; track title (Fraunces)
  + artist (accent); scrubber showing **elapsed and remaining** (`1:28 … −1:42`);
  transport row (shuffle, prev, **tan pause/play**, next, repeat); heart react.
- **Chat recedes, not gone:** the chat feed slides down into a **dimmed, non-interactive
  strip** at the bottom (reactions + hearts still visible so the room's energy is felt),
  with a "**⌃ Swipe up to bring the chat back**" affordance. Swiping up (or tapping)
  restores full chat; it auto-recedes again on the next play. This is the product's
  signature moment — the song owns the room but the crowd is still faintly there.
- **One sound source at a time (decision 2026-07-03):** pressing play on the secure song
  **auto-pauses the snippet**. The persistent bottom snippet bar stays visible but
  minimized to a paused handle (AirPlay + queue affordances remain); it does not play
  concurrently. Leaving the Listen act keeps the secure song paused; the snippet may
  resume on user action. Never two `<audio>` elements playing at once — enforce with a
  single shared "now playing" client store.

## Player UI states (AUDIO_PLAYER block)

| State | UI |
|---|---|
| canPlay, plays remaining | Play button + "▶ {playsUsed}/{playLimit} plays used" (LIMITED only) |
| Playing | glowing cover art, scrubber w/ elapsed + remaining, transport controls, heart; chat recedes to dimmed strip w/ "swipe up" |
| Limit reached | disabled player: "You've used your {n} plays. Full song out {releaseDate}." + Pre-save button if links exist |
| Not yet available | "Unlocks {availableFrom}" countdown |
| Expired | "This preview has ended." + Pre-save/streaming CTA |
| Owner preview | always playable, banner "Preview — plays don't count", no Play rows created (owner path skips token API, uses a signed URL server-side in the preview payload) |

## Explicitly deferred (do not build)

Encrypted HLS/ffmpeg pipeline, forensic watermarking, screen-recording detection,
DRM (Widevine/FairPlay), concurrent-stream caps.
