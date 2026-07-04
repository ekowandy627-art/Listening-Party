# 10 — Acceptance Checklist (final gate)

Run top to bottom against a fresh seed (`prisma migrate reset && prisma db seed`,
`pnpm dev`, MinIO + Postgres up). Every box must pass. Use two browsers (or a private
window) for artist vs fan.

## Auth (Module 1)

- [ ] `/join/victor/new-single` renders party card with cover, artist name, member count while signed out
- [ ] Entering an email sends a magic link (console in dev); `/auth/verify` shows the address
- [ ] Clicking the magic link creates the account, joins the party, and lands **inside** `/p/victor/new-single` — one click, no extra join step
- [ ] Same email via magic link twice → same account (no duplicate user)
- [ ] Google button appears only when configured; signing in with Google using an email that already exists links to that account
- [ ] `/admin` as non-admin → 404; `/studio` as anonymous → signin redirect

## Artist dashboard (Module 2)

- [ ] Fan account can "Become an artist" (handle availability live-checked) and reaches the studio
- [ ] Create party → draft appears; edit all detail fields incl. slug; join URL preview updates
- [ ] Publish blocked without cover/blocks with inline reasons; passes after fixing
- [ ] Duplicate creates a full DRAFT copy (blocks, tracks, links); archive hides from public; delete works on drafts only (409 otherwise)
- [ ] Status pills: DRAFT→PUBLISHED→LIVE→COMPLETED transitions all work with confirm dialogs

## Experience builder (Module 3)

- [ ] All 15 block types can be added, configured, and render correctly in preview
- [ ] Renaming an act via `actLabel` updates the act rails and storyboard
- [ ] Drag-and-drop reorder persists across reload
- [ ] Invalid block config shows inline error and blocks publish with per-block message
- [ ] Owner "Preview" opens the player with Preview ribbon regardless of status

## Secure audio (Module 4)

- [ ] Upload an MP3, set "Two plays", attach to AUDIO_PLAYER block
- [ ] Fan sees "0/2 plays used"; plays stream; seeking works (Range 206)
- [ ] Third play attempt → friendly limit screen (429 path), pre-save CTA shown
- [ ] Page source/network tab contains no R2 URL or fileKey; `/api/media/tracks/...` → 403
- [ ] `expiresAt` in the past → "This preview has ended"; `availableFrom` in future → countdown
- [ ] Owner preview plays without consuming plays
- [ ] Listen act enters focus mode: room dims, chat recedes to a swipe-up strip and restores on swipe
- [ ] Pressing play on the secure song auto-pauses the snippet bar (never two audios at once)
- [ ] Vitest: parallel play-token race test passes

## Fan experience (Modules 1+3)

- [ ] Room renders three-pane on desktop / single-scroll on mobile, in the party's mood preset
- [ ] Acts guided in order on first pass, then freely revisitable from the sidebar/act rail; unreached acts locked
- [ ] Poll: vote → % bars; revisit shows my vote; Question: answer saved, editable
- [ ] Countdown ticks live and shows doneText when past
- [ ] Reload mid-experience resumes at same block
- [ ] Finale fires `completedExperience=true` and shows Community/Share/Pre-save/Follow

## Community (Module 5)

- [ ] Fan and artist can post (artist badge visible); image attach works
- [ ] Comment, nested reply, likes on both posts and comments toggle with counts
- [ ] Artist can pin (pinned floats to top) and delete any post; fan only own
- [ ] Report → appears in admin Reports; resolve-and-delete removes the post
- [ ] Post search filters the feed; community still accessible on a COMPLETED party
- [ ] Non-member hitting community URL → join page
- [ ] Room Chat: two browsers in the same room see each other's messages within ~5 s (polling)
- [ ] Presence: online count reflects both browsers; drops after one goes idle >5 min
- [ ] A message posted in Room Chat appears in the Community feed (same stream) and vice versa

## Pre-save (Module 6)

- [ ] Artist saves Spotify + Apple links (bad host rejected with message)
- [ ] PRESAVE_CTA block shows both platform buttons; click opens link in new tab and flips to "Done ✓ / Reminder set"
- [ ] `Membership.preSavedAt` set on first click; pre-save rate on analytics reflects it

## Notifications (Module 7)

- [ ] go-live → members get PARTY_LIVE (bell + console email); release → SONG_RELEASED
- [ ] Artist post → ARTIST_POSTED to members, not to the artist
- [ ] Reply → COMMUNITY_REPLY to the right author only, in-app, no email
- [ ] Publish → NEW_PARTY to followers only
- [ ] Bell badge counts unread; mark-all-read clears; unsubscribe link stops emails (User.emailOptOut)

## Analytics (Module 8)

- [ ] After the fan journey above, analytics shows: funnel with real numbers, joins-over-time, per-track plays/unique/completion/replays, devices donut, referral sources incl. `ref:{name}`
- [ ] Every widget has a sane empty state on a brand-new party

## Sharing (Module 9)

- [ ] Share sheet: copy link (with my referral code), QR download (PNG scans to join URL)
- [ ] Join via someone's ref link → attributed on audience leaderboard with badge tiers
- [ ] Email invites (≤100) send and appear in Invited count

## Artist profile & search (Modules 10+11)

- [ ] `/a/victor` shows bio/genres/links, upcoming + past parties, working Follow
- [ ] Discover search finds artist by name/handle/genre and party by title; trending and featured rows populate
- [ ] `/me` lists joined parties with correct CTA (continue vs community) and notifications

## Admin (Module 12)

- [ ] Overview totals + sparklines render
- [ ] Ban a user → their existing session invalidated, signin blocked with message
- [ ] Feature flag `featured:{partyId}` puts the party in Discover's Featured row

## Non-functional

- [ ] Everything usable at 390 px; player and builder both tested on mobile viewport
- [ ] Signed-out public pages (`/`, `/discover`, `/a/*`, `/join/*`) render with no console errors
- [ ] Keyboard-only pass: signin, join, player navigation, posting all reachable
- [ ] `pnpm build` clean; `pnpm vitest run` green; no `fileKey` in client bundle grep
- [ ] OG tags: pasting a join URL into a link-preview checker shows cover art + title

## Deployment (run on the live production URL — doc 11)

- [ ] App is live on `{project}.vercel.app`; `/`, `/discover` render with no errors
- [ ] Magic-link sign-in with the owner's real email arrives via Resend (`onboarding@resend.dev` sender) and works
- [ ] Owner account has ADMIN role (prod seed); `/admin` loads
- [ ] "Challe from Ghana" artist profile exists at `/a/challefromghana`; real song file, cover art, and story text are added to the scaffolded party → published
- [ ] Room theme (mood preset/accent/background) set for the real party, not left on defaults
- [ ] Incognito session joins via the live join URL with a second email, walks the room, plays the track (Range streaming works on Vercel), posts in community
- [ ] Confirm: a fan email address *other than* the Resend account owner's does NOT receive the magic link (expected limitation, not a bug) — noted for the post-launch domain upgrade
- [ ] Play limit enforced on prod; no R2 URL/fileKey visible in prod network tab
- [ ] Analytics page shows the smoke-test activity (country now non-null via Vercel header)
- [ ] Notification email (e.g. go-live) actually delivered
- [ ] BUILD-NOTES.md records any fallbacks used (e.g. resend.dev sender restriction)
