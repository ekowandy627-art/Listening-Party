# 04 — Routes & Screens

**Design language is defined in `docs/12-theming.md` (the UI source of truth):** LP
logo, Inter (UI) + Fraunces (room emotional headings) + Caveat (artist signature only),
coral→amber flame gradient, warm
frosted-glass room surface, 6 mood presets. The earlier violet/fuchsia note is dead.
Every screen is mobile-first at 390 px; the room and studio get richer desktop layouts
at ≥1024 px.

Global chrome:
- **Public/fan shell**: top bar with logo, Discover link, and (signed-in) notification
  bell w/ unread badge + avatar menu (Me, Studio if artist, Sign out).
- **Studio shell**: left sidebar (Parties, Profile, back-to-fan-view) on desktop,
  bottom tabs on mobile.
- The **room** (`/p/...`) hides the public shell and uses its own three-pane chrome
  (see §4) — immersive, themed per the party's mood preset.

---

## 1. `/` — Landing (public)

Hero: "Turn your release into an experience." Sub: one-liner from PRD overview.
CTAs: "Create your listening party" (→ signin → studio) and "Explore parties"
(→ /discover). Below: 3-step how-it-works (Story → Listen → Community), then a
strip of trending party cards (reuse discover query). Footer: privacy, terms stubs.

## 2. `/discover` — Search & discovery (public)

Search input (debounced 300 ms → `GET /api/search`). Tabs: All / Artists / Parties.
Below the fold when query empty: **Featured** row (flag-driven), **Trending** grid,
**New** grid. Party card = cover art, title, artist name, status pill
(`Coming soon` PUBLISHED / `Live now` LIVE / `Released` COMPLETED), member count.
Artist card = photo, name, genres, follower count. Empty search state: "No results —
try an artist name or genre."

## 3. `/join/[handle]/[slug]` — Join page (public; THE viral surface)

Full-bleed cover art background (blurred) with join card:
- "Join **{Artist}**'s Listening Party" / party title / release countdown if future
  `releaseDate` (DAYS/HRS/MINS/SECS blocks) / social proof: overlapping avatar stack +
  "1,247 members already in" (+ "from 18 countries" when countryCount ≥ 2). Card uses
  the party's mood-preset tokens so the door already feels like the room.
- Signed out: email input + Continue (magic link, `callbackUrl` back into `/p/...`
  with `?ref` preserved), Google button (and Apple when configured), fine print
  "We'll email you a magic link — no password needed."
- Signed in, not a member: single button "Enter the party" → `POST /api/join/...`
  → redirect `/p/...`.
- Signed in + member: "Welcome back" → `/p/...`.
- Party DRAFT/archived: 404. COMPLETED: still joinable ("The song is out — join the community").
- `?ref=CODE` in URL is carried through the whole auth round-trip.

## 4. `/p/[handle]/[slug]` — The Room (member only, owner preview)

The core fan surface: a guided, themed **room**, not a full-screen story stack.
Approved desktop layout is **three panes + a persistent bottom player** (mockup 2026-07-03).

### Desktop (≥1024 px) — three panes

- **Left rail (act nav):** LP mark; party title + artist + cover thumbnail; the ordered
  list of **acts** (each block/act = one row with icon + fan-facing label from 05, e.g.
  Welcome · The Story · The Studio · Listen · Lyrics · Chat with {artist} · Pre-save ·
  Stay Connected); a thin progress bar; "Releases in" countdown; "Invite a friend"
  (opens share sheet). Act rows show state: **done ✓ / current (highlighted) / locked**.
- **Center stage:** the current act's content (block renderer from 05). The Welcome act
  = hero image, "YOU'RE IN" eyebrow, Fraunces room title, intro line + Caveat artist
  signature,
  "N fans are here early from M countries" proof pill, and the act's primary action
  (e.g. "Watch Welcome Video"). Below the stage, a **"What happens next"** storyboard
  strip: horizontally-scrolling numbered cards previewing upcoming acts (image, label,
  one-liner, duration/type icon).
- **Right rail (Room Chat):** header "Room Chat" + presence ("237 people here",
  online dot); message list (avatar, name, ARTIST badge for the artist, time, body,
  heart-react w/ count); pinned artist message; composer "Say something…". Near-live via
  **polling every 3–5 s** (`GET .../posts?mode=chat&after=` — Room Chat IS the party's
  Post stream, rendered chat-style; hearts = post likes; see 02/03). **Recedes during
  the Listen act** (fades/narrows so the song owns the room), returns after.
- **Bottom bar (persistent player):** a global mini-player looping the **snippet/teaser
  track** (freely playable, not the secure song). Cover, title "({Snippet})", artist,
  play/prev/next, scrubber, volume. This is the "music is always on" layer. **The full
  unreleased song is NOT here** — it lives only in the Listen act under play-limit
  security (docs/07). If the party has no snippet, the bar hides.

### Navigation model — guided first pass, free revisit

- **First pass is ordered:** the fan is walked act-by-act via the stage's primary action
  / "Continue"; the next act unlocks as each is reached. This preserves the funnel to
  Pre-save + community.
- **After an act is reached, it's freely revisitable** from the left rail (click any
  unlocked act). Not-yet-reached acts are locked (subtle, with a tooltip "Coming up").
- Progress (furthest act reached) persisted in `localStorage` (`lp-progress:{partyId}`)
  **and** server-side via the `experience_completed`/act events so it survives devices.
- Keyboard: ↑/↓ or ←/→ move between unlocked acts; Space toggles the current media.

### Mobile (<1024 px) — approved mockup, 2026-07-03

A single vertical scroll between a fixed top bar and a fixed bottom player:
- **Fixed top bar:** down-chevron (exit/minimize the room → back to `/me` or join),
  centered "{Artist}'s Room" + "● N here" presence, share, overflow (⋯).
- **Act stage (top of scroll):** full-bleed hero with a presence pill overlaid
  ("Ama, James +235 are here early"), amber act eyebrow ("WELCOME"), Fraunces heading
  ("Hey fam! 👋"), intro copy, the act's tan primary CTA ("Watch welcome video"), then a
  row of **progress dots** (one per act, current = amber).
- **"THE EXPERIENCE" rail:** a horizontally-scrolling row of **circular act icons**
  (Welcome · The Story · The Studio · Listen · Lyrics · Chat…), current one ringed in
  accent with its label; tap to switch acts (respecting locked/guided rules). This is the
  mobile act nav — always visible, no hamburger.
- **Room Chat (inline, below the rail):** pinned artist message then the feed
  (avatar, name, time, body, ❤ count). Scrolls within the page; a "jump to compose"
  affordance. During the **Listen** act the chat recedes into a **dimmed, non-interactive
  strip** (reactions still visible) with a "⌃ Swipe up to bring the chat back" handle;
  swipe/tap restores it, and it auto-recedes on the next play. See docs/07 "Listen act".
- **Fixed bottom snippet player:** compact bar ("{title} / {artist}", prev/pause/next,
  tan control) — the freely-playable snippet, persistent across acts.

### Finale & shared behavior

- Reaching the final act fires `POST .../complete` and shows the **Stay Connected / finale**
  state: "You made it!" (Fraunces), Share (referral link + native share + QR), Pre-save
  (if links), Follow {artist}, Enter Community.
- **Owner preview:** the party owner may open the room in any status with a "Preview"
  ribbon; all acts unlocked, plays don't count (docs/07).
- Analytics: `block_viewed` per act per session; media acts post `block_progress`
  (25/50/75/100); `share_click`, `presave_click`, `merch_click` as specced (08).
- Zero-act party (shouldn't pass publish validation): show the finale state directly.

## 5. `/p/[handle]/[slug]/community` — Community (member only)

Standard feed layout (chrome returns here):
- Composer at top (textarea, image attach via presign, 2000-char counter).
- Pinned posts first with pin badge; artist posts get an "Artist" badge + subtle
  gradient border.
- Post card: avatar, name, time-ago, body (linkify), image, like button w/ count,
  comment count → expands inline comment thread (one nesting level, reply buttons),
  overflow menu (report; delete for author/artist/admin; pin for artist).
- Search field filters posts (`q` param). Cursor "Load more".
- Non-members hitting this URL → redirect to join page.

## 6. `/a/[handle]` — Artist profile (public)

Banner-less hero: photo, name, genres chips, bio, links (external icons),
Follow/Following button (signin-gated), follower count.
Sections: **Upcoming** (PUBLISHED/LIVE parties as cards → join pages),
**Past parties** (COMPLETED → join pages, which route members into community).
Empty states: "No upcoming releases — follow to be first to know."

## 7. `/me` — Fan home (session)

- "Your parties": membership cards (cover, title, artist, status pill, CTA
  "Continue experience" if `!completedExperience` else "Open community").
- "Following": artist chips.
- Notifications list (same data as bell dropdown, full page): unread highlighted,
  click → `url` + mark read. "Mark all read".
- Account section: name/avatar edit, sign out, danger zone (delete account w/ confirm
  dialog typing DELETE).
- If user has an ArtistProfile: banner link "Go to Studio".

## 8. `/auth/signin` + `/auth/verify`

Signin: email → magic link (POSTs to Auth.js), Google/Apple buttons, honors
`callbackUrl`. Verify: "Check your email" with the address shown and "use a
different email" link. Both centered cards on dark background.

## 9. `/studio` — Artist dashboard (ARTIST)

- If session user has no ArtistProfile: **Become an artist** card — name + handle
  (live availability check) + genres → `POST /api/artists` → dashboard.
- Dashboard: "New Listening Party" primary button. Party table/cards: cover thumb,
  title, status pill, release date, members, pre-saves; row actions: Edit, Audience,
  Analytics, Duplicate, Archive/Unarchive, Delete (drafts only, confirm dialog).
  Filter tabs: Active / Drafts / Archived. Empty state with illustration + CTA.

## 10. `/studio/parties/new`

Minimal create form: title (required), description, genre (free text w/ suggestions),
release date (optional datetime). Submit → `POST /api/parties` → redirect to edit.

## 11. `/studio/parties/[id]/edit` — Details + Experience Builder (owner)

Two tabs:

**Details tab** — form: title, slug (editable, shows full join URL preview), description,
genre, release date, cover art upload (square crop preview), banner upload, pre-save
links editor (one URL input per platform with platform icons; validated on save →
`PUT .../presave-links`).

**Experience tab** — the builder:
- Left (mobile: top) — **block list**: dnd-kit sortable cards showing type icon +
  type name + one-line summary of config. Drag to reorder (`POST .../blocks/reorder`
  on drop). Click selects.
- "Add block" button → sheet/grid of the 15 block types with icons + descriptions;
  every block's editor also exposes the optional **act label** field (05).
- Right (mobile: sheet) — **block editor**: form per type (05 defines every field),
  autosaves on blur/change with 800 ms debounce (`PATCH /api/blocks/[id]`), delete
  button w/ confirm.
- **Preview** button → opens `/p/...` in preview mode: owner may always view their own
  party player regardless of status/membership (owner bypass in the player's access
  check), with a "Preview" ribbon.
- Header status area: current status pill + context action:
  DRAFT → "Publish" (runs validations; failures listed inline: needs title/cover/≥1 block)
  PUBLISHED → "Go live" + "Mark released"; LIVE → "Mark released"; COMPLETED → none.
  Each transition has a confirm dialog stating which notifications will be sent.

## 12. `/studio/parties/[id]/audience` (owner)

- **Share card**: join URL w/ copy button, QR code image (`/api/parties/[id]/qr`,
  download button), "Referral links are unique per fan — yours is below."
- **Invite by email**: textarea (comma/newline separated, ≤100), send → toast with count.
- **Members table**: name/email, joined date, referred-by code, completed ✓,
  pre-saved ✓. Search + CSV export (client-side from loaded data, `members.csv`).
- **Referral leaderboard**: top codes by joins (from `Membership.referredByCode`
  grouped), showing member name + join count + badge tier (🥉 3+, 🥈 10+, 🥇 25+).

## 13. `/studio/parties/[id]/analytics` (owner)

Renders the `GET /api/parties/[id]/analytics` payload (08):
- Stat cards row: Invited, Joined, Completed experience (+% of joined), Pre-save rate,
  Community members, Comments.
- **Funnel** bar: invited → joined → completed → pre-saved.
- **Listening**: per-track card — plays, unique listeners, avg completion %, replay
  count (plays beyond first per user); "most replayed" sort.
- **Watch time**: avg completedPct across video blocks (from block_progress events).
- Charts: joins over time (line, 30 d), countries (top-8 bar), devices (donut),
  referral sources (top referrers list: referral codes + external referrer hosts).
- Empty/low-data states: "Not enough data yet" per widget, never broken charts.

## 14. `/studio/profile` (owner)

Edit ArtistProfile: photo upload, name, handle (with rename warning that old links
break), bio, genres (chip input), links (label+URL rows, add/remove). Public profile
link out.

## 15. `/admin` (ADMIN)

Single page, four tabs:
- **Overview**: totals + 30-day joins/plays sparklines.
- **Users**: search table, role select, ban toggle (confirm).
- **Reports**: open reports w/ post body preview → Resolve (keep) / Resolve & delete post.
- **Flags**: key/enabled/note rows, add flag. Convention `featured:{partyId}` drives
  the Discover featured row.

## 16. System pages

`/privacy`, `/terms` — static stubs. `not-found.tsx` and `error.tsx` — branded,
with "Back to Discover". Loading states: skeleton cards on every data page
(`loading.tsx` per route group).
