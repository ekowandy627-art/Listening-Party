# 12 — Visual System & Theming (source of truth for UI)

This doc is derived from the approved brand guide, logo, and the 13-screen mockup set
(2026-07). It supersedes the placeholder palette mentioned earlier in docs/04. Build
the UI to these tokens.

## Two surfaces, one brand

| Surface | Where | Mood | Reference |
|---|---|---|---|
| **Brand surface** | Marketing (`/`, `/discover`, `/a/*`), Studio, Admin, auth cards | Clean, confident, professional. Inter, coral→amber accents, structured. | Brand guide |
| **Room surface** | The experience (`/p/...`), join page, finale, fan `/me`, community | Warm, candle-lit, atmospheric, intimate. Frosted dark panels floating over a lit-room background; handwritten headings for emotional beats. | Mockups 1–7 |

Same components underneath; the surface decides the tokens.

## Logo

LP monogram — black **L** + coral→amber **P** whose bowl contains a play triangle;
wordmark "LISTENING PARTY". Variants: full-color, black, white (knockout), icon-only.
Store in `app/public/brand/` as: `lp-full.svg`, `lp-mono-black.svg`, `lp-mono-white.svg`,
`lp-icon.svg`. Rule: clear space = height of the "L"; never stretch, recolor, or add
effects. In the room (dark) use the white or icon-only mark; on light marketing use full-color.

## Color tokens

### Brand palette (from the guide)

```
--brand-ink:        #111111   /* base black — focus, sophistication */
--brand-coral:      #FF5A3C   /* gradient start */
--brand-amber:      #FFB347   /* gradient end */
--brand-flame:      linear-gradient(135deg, #FF5A3C 0%, #FFB347 100%); /* primary CTA + logo */
--brand-mist:       #F5F5F5

--grey-900: #222831
--grey-500: #6B7280
--grey-400: #9CA3AF
--grey-200: #E5E7EB
--white:    #FFFFFF
```

Usage: **flame gradient = primary action + brand energy**, used sparingly (main CTAs,
logo, key highlights). Black is the base. Greys keep the interface quiet and out of the way.

### Room tokens (Warm Lounge default — the hero look)

The room is not flat dark; it's a *lit space*. Backgrounds are warm imagery/video; UI
is frosted charcoal glass floating over it.

```
--room-bg:          #14100C   /* warm near-black behind the background image */
--room-glow:        #E8A24A   /* the "lighting colour" — amber candlelight, per-preset */
--panel:            rgba(28, 22, 16, 0.72)      /* frosted card fill */
--panel-blur:       backdrop-filter: blur(20px) saturate(120%)
--panel-border:     rgba(255, 245, 230, 0.10)
--panel-radius:     20px
--room-heading:     #F6EEE1   /* warm off-white */
--room-text:        #D8CDBE
--room-muted:       #9C907F
--room-cta:         #CBA06A   /* soft tan/gold button (mockups) — the room's gentle CTA */
--room-cta-text:    #1B140D
```

Rule of thumb: **flame gradient for high-intent conversion moments (pre-save, "Enter
the room"); soft tan `--room-cta` for gentle in-room progression ("Continue," "Play
video").** Never put the loud gradient on quiet moments — it breaks the mood.

## Typography

```
--font-system:  "Inter"           /* ALL UI, body, marketing, studio, chat, buttons */
--font-display: "Fraunces"        /* room emotional headings: "Hey fam!", act titles, finale */
--font-script:  "Caveat"          /* artist SIGNATURE line only ("– Victor") */
```

Confirmed 2026-07-03 from the hi-fi room mockups: **serif display for emotional
headings, script only for the signature.**

- **Inter** (Bold / Semibold / Medium / Regular) — everything functional and all
  brand-surface headings, per the brand guide. Hero headings on marketing = Inter Bold/tight.
- **Fraunces** (a warm high-contrast display serif; opsz/soft settings on) — the room's
  *feeling* headings: the Welcome "Hey fam! 👋", the room title ("Victor's Listening
  Room"), act titles ("The Story," "Listen"), the finale ("You made it!"), auth room
  cards ("Welcome back," "Check your email"). Large, tight, emotional. Do not use for UI,
  labels, or anything you tap.
- **Caveat** (handwritten script) — used **only** for the artist's signature line
  ("– Victor") beneath an intro/message. Nowhere else.
- **Lyrics** act: Inter (or Fraunces for the title), large, centered, editorial spacing
  (stanzas separated by blank lines), subtle top/bottom gradient fade.

Load all three via `next/font/google`.

## Mood presets (screen 12 — Room Theme)

Artist picks one preset (sets `--room-glow`, base background tone, surface warmth,
default accent) then may override the **accent color** via a swatch picker and swap the
**background image/video** and **intro message**. Six presets:

| Preset | Glow / lighting | Background feel | Default accent |
|---|---|---|---|
| **Warm Lounge** (default) | Amber candlelight `#E8A24A` | Cozy lamp-lit room, string lights | Tan/gold |
| **Neon Night** | Magenta/violet `#B65CFF` | Dark club, neon haze | Electric violet |
| **Ocean Depths** | Teal `#3FB6C4` | Deep blue underwater light | Cyan |
| **Forest Echo** | Green `#5FA86B` | Misty forest, dappled light | Moss |
| **Sunset Drive** | Warm orange `#FF7A3C` | Dusk highway, gradient sky | Coral |
| **Minimal White** | Soft neutral | Clean light room (the only light room theme) | Brand coral on white |

Implementation: a preset is a small JSON on the party (`Party.theme Json`) —
`{ preset, accent, backgroundKey?, backgroundVideoKey?, intro }`. Presets map to a CSS
variable set applied on the room's root (`data-preset="warm-lounge"`). Minimal White
flips the room to light surfaces (dark text on frosted white). Background video always
needs a poster image and pauses during the Listen act.

> **Schema addition:** add `theme Json @default("{}")` to the `Party` model (docs/02).
> The build session applies this in the initial migration.

## Desktop room layout (approved mockup, 2026-07-03)

The room on desktop (≥1024 px) is a **three-pane app-like layout with a persistent
bottom player**, not a full-screen story. See docs/04 §4 for behavior; the visual shape:

```
┌────────────┬───────────────────────────────┬──────────────┐
│ LEFT RAIL  │        CENTER STAGE           │  RIGHT RAIL  │
│ LP mark    │  current act (hero/media)     │  Room Chat   │
│ title+cover│  "YOU'RE IN" / Caveat title   │  presence    │
│ ACT NAV    │  intro + signature            │  messages    │
│  ✓ Welcome │  proof pill · primary action  │  ARTIST badge│
│  ▸ Story   │                               │  ❤ reactions │
│    Studio  │  ── "What happens next" ──     │  pinned msg  │
│    Listen  │  [1][2][3][4]… storyboard      │  composer    │
│    …       │                               │              │
│ countdown  │                               │              │
│ Invite     │                               │              │
├────────────┴───────────────────────────────┴──────────────┤
│  ▸ persistent bottom player — SNIPPET only, not secure song │
└─────────────────────────────────────────────────────────────┘
```

- Left/right rails are **light brand-surface on desktop** in mockup 14, but tint toward
  the room mood; center + bottom carry the atmospheric room treatment. Keep AA contrast.
- Act nav rows: icon + label, states done ✓ / current (accent highlight) / locked (dim).
- Storyboard cards: numbered, cover thumb, label, one-liner, duration/type icon.
- Bottom player: Spotify-familiar controls; label reads "{title} (Snippet)".
### Mobile room (approved mockup, 2026-07-03)

Single vertical scroll between a fixed top bar and a fixed bottom snippet player:

```
┌─────────────────────────────┐
│ ⌄   Victor's Room   ⤴  ⋯    │ fixed top bar (exit · title+presence · share · more)
├─────────────────────────────┤
│  [avatars] +235 here early   │ presence pill over hero
│                             │
│      full-bleed hero        │
│                             │
│         WELCOME             │ amber act eyebrow
│      Hey fam! 👋            │ Fraunces heading
│   intro copy…               │
│   [ Watch welcome video ▸ ] │ tan act CTA
│      • • • • • • • •         │ act progress dots
│  ── THE EXPERIENCE ──        │
│  (◉)  ▢   ▢   ▢   ▢   ▢ →    │ horizontal circular act rail
│  Welcome Story Studio Listen │
│  ┌ Ama · Pinned    ❤12 ┐    │ inline room chat feed
│  └ James …               ┘  │
├─────────────────────────────┤
│ ▸ New Chapter · Victor  ⏸    │ fixed bottom snippet player
└─────────────────────────────┘
```

- Act rail is a horizontal scroll of circular icons (no hamburger); current = accent ring + label.
- Chat is inline in the page scroll (not a separate sheet); recedes during Listen.

### The Listen act — focus mode (signature screen, mockup 2026-07-03)

The peak moment. Entering Listen dims everything but the song:

```
┌─────────────────────────────┐
│ ⌄  Victor's Listening Room  ⋯│
│        ◍ THE LISTEN ACT      │ eyebrow pill
│  Be the first to hear.       │ instruction
│  Chat will return after…     │
│     ┌───────────────┐        │
│     │  glowing cover │        │ large square art, warm glow
│     └───────────────┘        │
│      New Chapter        ♥     │ Fraunces title
│      Victor Ray              │ accent artist
│  1:28 ●━━━━━━━━━     −1:42    │ elapsed + remaining
│   ⤨   ⏮   (⏸)   ⏭   ⟳       │ tan pause hero
│      ⌃ Swipe up for chat     │
│  ┌ Ama  The intro… ✨   ❤12 ┐│ DIMMED chat strip
│  └ James  That bass…   ❤ 9 ┘│ (non-interactive)
├─────────────────────────────┤
│ ▸ New Chapter · Victor  ⏸ ≣ │ snippet bar → paused handle
└─────────────────────────────┘
```

- Everything except the player desaturates/dims; the cover art glow is the only warm
  focal point. Motion stills (no storyboard, no bouncing).
- Chat strip is dimmed + non-interactive with a swipe-up handle; restores on gesture,
  auto-recedes on next play.
- The persistent snippet bar shows **paused** (one sound source rule, docs/07).
- Respect `prefers-reduced-motion` (cross-fade the dim, don't animate).

## Component styling notes (from the mockups)

- **Cards/panels:** `--panel` fill, `--panel-blur`, `--panel-border`, `--panel-radius`,
  soft drop shadow. Everything floats; no hard 1px lines. This frosted-glass panel is
  *the* signature component — join card, welcome card, player, chat rail, community posts.
- **Countdown (join):** four stacked blocks DAYS/HRS/MINS/SECS, panel-styled, large Inter.
- **Member proof:** overlapping circular avatar stack + "1,247 members already in."
- **Audio player (Listen act):** centered cover art with warm glow, big circular
  play/pause, scrubber, prev/next, heart; "You have 2 listens remaining" line beneath.
  During playback the room dims and chat recedes (docs/04).
- **Chat rail (desktop):** right-docked frosted panel, "Room Chat / 124 online," avatar +
  name + message rows, input pinned bottom ("Say something…"). Mobile: inline feed in
  the page scroll (see mobile room layout above); recedes to the dimmed strip during Listen.
- **Community feed:** tabs (All / Announcements / Talk / Media), pinned artist post with
  badge, like/reply counts, bottom tab bar (home / gallery / likes / notifications / profile).
- **Studio:** left sidebar (Dashboard, My Parties, Audience, Analytics, Profile, Settings),
  clean brand-surface styling, party rows with status + member/presave counts, flame-gradient
  "New Listening Party" button. Experience builder = left act list (numbered) + right editor
  + "Preview room."
- **Analytics:** stat cards (Invited / Joined / Completed / Pre-saved), joins-over-time
  line (coral), top countries + donut, top sources list.
- **Motion:** slow eased cross-fades between acts; the room "breathes" (subtle glow pulse
  on cover); stillness during Listen. Respect `prefers-reduced-motion`.

## Icon & imagery style

Simple, friendly line icons (play, headphones, community, chat, heart, bell, share,
analytics) — consistent stroke, per brand guide. Imagery: high-contrast, cinematic,
*warm* photography of real artists/fans — silhouettes against stage light, crowds, close
candid moments. Never stock "abstract 3D blobs."

## Tone of voice (applies to all copy)

Authentic ("Real conversations. Real connections."), Empowering, Exciting, Trustworthy.
First-person from the artist inside the room ("This room is for you," "Sit with this
one," "You're all set!"). Never corporate ("Submit," "Success").

## Accessibility guardrails on the atmosphere

- Frosted panels must keep text at **AA contrast** over any background image — enforce a
  minimum panel opacity/scrim so headings stay ≥ 4.5:1. Test each preset.
- Handwritten Caveat is decorative-scale only (large sizes); never body or small text.
- The whole room must be usable with motion disabled and with a screen reader (act
  titles are real headings, player has labeled controls, chat is a live region).
