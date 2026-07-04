# 08 — Notifications & Analytics

## Notifications

Two channels, always written together by `lib/services/notify.ts`:
1. **In-app**: `Notification` row (drives the bell + `/me`).
2. **Email**: via `lib/email.ts`. Skipped when the recipient's
   `Membership.emailOptOut` is set (party-scoped mute) **or** `User.emailOptOut` is
   set (global unsubscribe — the email-footer link sets this; both fields are in 02).
   Check both flags before any send.

`notifyMembers(partyId, type, payload, { excludeUserId })` and
`notifyFollowers(artistId, …)` fan out with `Promise.allSettled` in batches of 100.
Failures are logged, never thrown to the caller.

| Trigger (API) | Type | Recipients | Email subject |
|---|---|---|---|
| `publish` | `NEW_PARTY` | artist's followers | `{Artist} just announced {title}` |
| `go-live` | `PARTY_LIVE` | party members | `{title} is live — come in` |
| `release` | `SONG_RELEASED` | party members | `{title} is OUT NOW 🎉` (body includes streaming/pre-save links) |
| artist creates post | `ARTIST_POSTED` | party members (excl. artist) | `{Artist} posted in {title}` |
| comment on your post / reply to your comment | `COMMUNITY_REPLY` | post/comment author (excl. self) | in-app only, **no email** (too noisy for MVP) |

Every email footer: unsubscribe link (`/api/unsubscribe?token=` — HMAC-signed userId,
one click, no login needed) + "You're getting this because you joined {party} on
Listening Party."

Emails are simple branded HTML (logo text, dark-on-light for deliverability, one CTA
button) built with plain template strings — **no react-email dependency**.

## Analytics

### Event taxonomy (`AnalyticsEvent.type`)

Server-emitted (inside services — authoritative):
| type | meta | emitted by |
|---|---|---|
| `user_signed_up` | `{}` | Auth.js event |
| `invite_sent` | `{ email }` | invites API |
| `party_joined` | `{ ref?, source? }` | join service |
| `experience_completed` | `{}` | complete API |
| `presave_click` | `{ platform }` | presave-click API |
| `play_started` | `{ trackId }` | play-token API |
| `post_created` / `comment_created` | `{ postId }` | community service |

Client-emitted via `POST /api/events` (allowlist these types only, require session,
attach partyId):
| type | meta |
|---|---|
| `join_page_view` | `{ ref? }` — *exception: no session required; this is the top-of-funnel* |
| `block_viewed` | `{ blockId, index }` |
| `block_progress` | `{ blockId, pct }` (25/50/75/100, video+audio) |
| `share_click` | `{ channel: "copy"\|"native"\|"qr" }` |
| `merch_click` | `{ blockId }` |

Server derives on every event: `country` (`x-vercel-ip-country` header, null locally),
`device` (UA → mobile/tablet/desktop via simple regex, no library), `referrer`
(`document.referrer` passed in body for client events; `referer` header for
join_page_view).

### `GET /api/parties/[id]/analytics` response shape

```ts
{
  totals: { invited, joined, completed, preSaved, communityMembers, comments, shares },
  funnel: { invited, joined, completed, preSaved },          // same numbers, for the bar
  rates:  { joinRate, completionRate, preSaveRate },          // 0–1, null when denominator 0
  joinsOverTime: { date: string, count: number }[],           // last 30 days, zero-filled
  listening: [{ trackId, title, plays, uniqueListeners, avgCompletionPct, replays }],
  watchTime: { avgVideoCompletionPct: number | null },
  countries: { country, count }[],                            // top 8
  devices:   { device, count }[],
  referrals: { source, count }[],   // top 10: "ref:{memberName}" for codes, else referrer host, else "direct"
}
```

Computed with grouped Prisma/raw SQL queries at request time — no pre-aggregation,
no cron. `invited` = count of `invite_sent` events; `joined` = memberships;
`completed` = memberships with flag; `preSaved` = memberships with `preSavedAt`;
`shares` = `share_click` events; `communityMembers` = joined (community == party);
`replays` = total plays − unique (track,user) pairs.
