# 05 — Block System (Acts)

Blocks are the heart of the product. A party's experience is an ordered list of
blocks; **each block is an "act" in the room** — it appears as a row in the desktop
act-nav rail / a circle in the mobile act rail, and renders as the room's center
stage when current (docs/04 §4). Each has an editor form in the studio.

## Type definitions (`src/types/blocks.ts`)

Implement as a zod discriminated union on `type`. `BlockConfig` below is the `config`
JSON stored on the `Block` row. All media fields store **R2 keys**, resolved to signed
URLs server-side when the room payload is built (except audio — see AUDIO_PLAYER).

```ts
export const blockTypes = [
  "WELCOME_VIDEO", "VIDEO_STORY", "PHOTO_GALLERY", "LYRICS", "VOICE_NOTE",
  "TEXT", "COUNTDOWN", "POLL", "QUESTION", "AUDIO_PLAYER", "ARTIST_MESSAGE",
  "MERCH_PROMO", "PRESAVE_CTA", "CHAT", "COMMUNITY_INVITE",
] as const; // 15 types
```

**Shared field:** every block config also accepts `actLabel?: string (1..24)` — the
fan-facing name shown in the act rails and storyboard. When omitted, the per-type
default applies:

| Type | Default act label | Act icon |
|---|---|---|
| WELCOME_VIDEO | Welcome | home |
| VIDEO_STORY | The Story | book |
| PHOTO_GALLERY | The Studio | camera |
| LYRICS | Lyrics | quote |
| VOICE_NOTE | Voice Note | mic |
| TEXT | The Story | book |
| COUNTDOWN | Countdown | clock |
| POLL | Your Take | bar-chart |
| QUESTION | Ask {artist} | message-circle |
| AUDIO_PLAYER | Listen | waveform |
| ARTIST_MESSAGE | From {artist} | mail |
| MERCH_PROMO | Merch | shopping-bag |
| PRESAVE_CTA | Pre-save | heart |
| CHAT | Chat with {artist} | chat |
| COMMUNITY_INVITE | Stay Connected | users |

(This is how the mockups' "The Studio," "Chat with Victor," etc. exist without a page
builder — artists rename acts, never design them.)

| Type | Config schema (zod) | Editor fields | Player rendering |
|---|---|---|---|
| `WELCOME_VIDEO` | `{ videoKey: string, caption?: string }` | video upload, caption | Stage-filling video, autoplay muted w/ tap-to-unmute, caption overlay bottom. Continue enabled immediately (skippable) |
| `VIDEO_STORY` | `{ videoKey: string, title?: string, caption?: string }` | video upload, title, caption | Same player as welcome, title overlay top |
| `PHOTO_GALLERY` | `{ photos: { key: string, caption?: string }[] (1..12), title?: string }` | multi-image upload, per-photo caption, drag order | Swipeable carousel with dots; pinch-zoom not required |
| `LYRICS` | `{ title?: string, body: string (1..5000) }` | title, textarea (plain text, blank-line = stanza break) | Scrollable centered stanzas, large serif type, subtle gradient fade top/bottom |
| `VOICE_NOTE` | `{ audioKey: string, title?: string, durationSec?: number }` | audio upload (recorded voice memo etc.), title | "Voice note from {artist}" card: waveform-style progress bar (fake waveform = seeded random bars), play/pause. Uses plain signed URL (voice notes are not protected tracks) |
| `TEXT` | `{ title?: string, body: string (1..4000), align?: "left"\|"center" }` | title, textarea (markdown: bold/italic/links only), align | Rendered markdown, max-w-prose |
| `COUNTDOWN` | `{ title?: string, target: string (ISO datetime), doneText?: string }` | title, datetime picker (defaults to party releaseDate), done text | Live DD:HH:MM:SS flip-style counter; shows `doneText` ("It's out now 🎉") when past |
| `POLL` | `{ question: string, options: string[] (2..6), showResults: "afterVote"\|"always" }` | question, option rows add/remove | Options as buttons; after vote (`POST /api/blocks/[id]/vote` → PollVote upsert) show % bars + own choice highlighted. One vote per user, changeable |
| `QUESTION` | `{ prompt: string, placeholder?: string }` | prompt, placeholder | Prompt + textarea + submit (`POST /api/blocks/[id]/answer`, upsert). After submit: "Sent to {artist} 💌" + shows their answer, editable |
| `AUDIO_PLAYER` | `{ trackId: string, showLyricsBlockId?: string, note?: string }` | select from party's tracks (link to Tracks manager to upload), optional note | **The protected player** — see 07. Cover art spinning-vinyl treatment, play limit indicator ("2 plays left"), progress bar (no scrub-back restriction), no download/context menu |
| `ARTIST_MESSAGE` | `{ body: string (1..2000), photoKey?: string, signature?: string }` | textarea, optional photo, signature line | Letter-style card: artist avatar, message in larger italic-leaning type, signature |
| `MERCH_PROMO` | `{ title: string, imageKey?: string, description?: string, url: string (https), buttonText?: string (default "Shop") }` | fields as listed | Product card + external-link button (`target=_blank rel=noopener`; fires `merch_click` event) |
| `PRESAVE_CTA` | `{ headline?: string (default "Pre-save {title}"), subtext?: string }` | headline, subtext | Platform buttons from party's PreSaveLinks (icon + name). Click → `presave-click` API + open URL. After any click, block state shows "Done ✓" + "Reminder set for release day". If party has no links: block auto-skips in player, editor shows warning |
| `CHAT` | `{ prompt?: string (default "Say hi — {artist} is here"), pinnedMessage?: string }` | prompt, optional pinned message | The **Chat act**: brings the Room Chat center-stage — desktop widens the chat rail into the stage; mobile scrolls/expands the inline chat with the composer focused. `prompt` renders as the stage heading; `pinnedMessage` posts (once) as a pinned artist post when the party publishes. No new data model — it spotlights the same Post stream (02) |
| `COMMUNITY_INVITE` | `{ headline?: string (default "Join the conversation"), subtext?: string }` | headline, subtext | Shows 3 latest community posts as teaser cards + member avatars stack + "Join the community" → community page |

## Cross-cutting player rules

- **Access**: player payload endpoint is `GET /api/parties/[id]/experience` (member or
  owner). It returns blocks in order with media keys already resolved to signed URLs,
  poll/question state for the current user, track status summaries, and party/artist
  header data. One round-trip renders the whole player.
- **Interactive blocks** (POLL, QUESTION) allow skipping without answering — Continue
  is always available.
- **Media blocks** preload the next block's media when the current one becomes active.
- **Progress events**: video and audio blocks post `block_progress` analytics events at
  25/50/75/100% thresholds (`meta: { blockId, pct }`).
- **Vote/answer endpoints** (add to 03's list): `POST /api/blocks/[blockId]/vote`
  `{ optionIdx }` and `POST /api/blocks/[blockId]/answer` `{ answer (1..2000) }` —
  auth `member`, validate the block belongs to a party the user is a member of and
  the block type matches. `GET` of results comes bundled in the experience payload
  (poll counts per option; own answer).

## Editor UX rules

- New block gets sensible default config (e.g. POLL: question "", two empty options)
  and is selected immediately.
- Config forms validate with the same zod schemas; invalid fields show inline errors
  and don't autosave until valid.
- Block list card summaries: WELCOME_VIDEO → "Welcome video", POLL → question text
  truncated, AUDIO_PLAYER → track title, etc. — always give the artist a scannable
  storyboard of the experience.
- Publish validation requires every block's config to pass its zod schema (surface
  per-block errors in the publish dialog).
