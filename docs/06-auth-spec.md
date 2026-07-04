# 06 — Authentication Spec

## Stack

Auth.js (NextAuth v5) with the Prisma adapter, **database sessions** (not JWT) so
admin bans can revoke instantly. Session cookie: 30-day rolling expiry.

Providers:
1. **Email (magic link)** — primary path. Custom `sendVerificationRequest` uses
   `lib/email.ts` (Resend; console fallback in dev — print the full magic-link URL
   with a loud `📧 MAGIC LINK:` prefix so the builder/tester can click it from the
   terminal).
2. **Google** — enabled when `AUTH_GOOGLE_ID/SECRET` present.
3. **Apple** — enabled when `AUTH_APPLE_ID/SECRET` present; UI hides the button otherwise.

`allowDangerousEmailAccountLinking: true` for Google/Apple — email is the identity
key per the PRD, so a magic-link user later using Google with the same email must land
in the same account.

## Magic-link email

Subject: `Your link to {partyTitle}` when initiated from a join page (pass party title
through the signin call), else `Sign in to Listening Party`. Body: one big button
("Enter the party" / "Sign in"), plain-URL fallback, "expires in 24 hours, one-time
use" note. Sender: `EMAIL_FROM`.

## The join flow (must work exactly like this)

1. Fan hits `/join/victor/new-single?ref=AB12CD34` (signed out).
2. Enters email → client calls `signIn("resend"…, { email, callbackUrl:
   "/p/victor/new-single?ref=AB12CD34", redirect: false })` → route to `/auth/verify`.
3. Fan clicks emailed link → Auth.js verifies token, creates User (if new) + session,
   redirects to the callbackUrl.
4. `/p/...` page's server component sees a session but **no membership** → it
   auto-joins (server-side call to the join service with the `ref` param) instead of
   bouncing back to the join page. This makes the emailed link a true one-click entry.
5. `ref` is validated (existing `Membership.referralCode` in the same party); invalid
   codes are silently dropped.

Google path from the join page: same callbackUrl mechanics via OAuth `state`.

## Session shape & helpers

Extend the session callback to include `user.id`, `user.role`, and
`user.artistHandle` (nullable). Helpers in `lib/auth.ts`:

```ts
auth()                    // Auth.js session getter
requireUser()             // throws 401 Response
requireArtist()           // 403 unless role ARTIST/ADMIN, returns profile
requireMember(partyId)    // 403 unless Membership exists (or user owns the party, or admin)
requirePartyOwner(partyId)// 403 unless owns via ArtistProfile (admin bypasses)
requireAdmin()
```

All API routes and server actions use these — never inline session checks.

## signIn callback rules

- Reject when `user.banned` → Auth.js error page with "This account is disabled."
- On first-ever sign-in (user created), fire `user_signed_up` analytics event.

## Role transitions

- Everyone starts FAN.
- `POST /api/artists` (Become an artist) sets role ARTIST atomically with profile
  creation. Artists keep all fan abilities.
- ADMIN only via seed/DB or admin users tab.

## What is deliberately NOT built (final)

Passwords (PRD "later"), Spotify login (Future), email change, 2FA, rate limiting
beyond Auth.js' built-in token semantics — except: the magic-link request endpoint
gets a naive in-memory limiter (max 3 sends per email per 10 min) to prevent
accidental spam loops in demos.
