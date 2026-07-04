# 02 — Data Model (Prisma schema, copy verbatim)

This is the complete `prisma/schema.prisma` for the MVP. Copy it as-is; only the
Auth.js adapter models (`Account`, `Session`, `VerificationToken`) follow the
standard `@auth/prisma-adapter` shape.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ───────────────────────── Auth.js standard models ─────────────────────────

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

// ───────────────────────────────── Core ────────────────────────────────────

enum Role {
  FAN
  ARTIST
  ADMIN
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  emailVerified DateTime?
  name          String?
  image         String?
  role          Role      @default(FAN)
  banned        Boolean   @default(false)
  emailOptOut   Boolean   @default(false) // global unsubscribe (see 08)
  createdAt     DateTime  @default(now())

  accounts       Account[]
  sessions       Session[]
  artistProfile  ArtistProfile?
  memberships    Membership[]
  posts          Post[]
  comments       Comment[]
  likes          Like[]
  notifications  Notification[]
  follows        Follow[]        @relation("follower")
  plays          Play[]
  pollVotes      PollVote[]
  questionAnswers QuestionAnswer[]
  reports        Report[]        @relation("reporter")
}

model ArtistProfile {
  id        String   @id @default(cuid())
  userId    String   @unique
  handle    String   @unique // lowercase [a-z0-9-]
  name      String   // display/stage name
  bio       String?  @db.Text
  photo     String?  // R2 key
  genres    String[]
  links     Json     @default("[]") // [{label, url}]
  createdAt DateTime @default(now())

  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  parties   Party[]
  followers Follow[] @relation("followed")
}

model Follow {
  followerId String
  artistId   String
  createdAt  DateTime      @default(now())
  follower   User          @relation("follower", fields: [followerId], references: [id], onDelete: Cascade)
  artist     ArtistProfile @relation("followed", fields: [artistId], references: [id], onDelete: Cascade)

  @@id([followerId, artistId])
}

// ─────────────────────────────── Parties ───────────────────────────────────

enum PartyStatus {
  DRAFT
  PUBLISHED // joinable, pre-release
  LIVE      // release day / release moment triggered
  COMPLETED // post-release; community stays open
}

model Party {
  id           String      @id @default(cuid())
  artistId     String
  slug         String      // unique per artist
  title        String
  description  String?     @db.Text
  coverArt     String?     // R2 key
  banner       String?     // R2 key
  genre        String?
  releaseDate  DateTime?
  theme        Json        @default("{}") // { preset, accent, backgroundKey?, backgroundVideoKey?, intro } — docs/12
  snippetKey   String?     // R2 key of the freely-playable teaser for the bottom player (docs/07); NOT play-limited
  status       PartyStatus @default(DRAFT)
  archivedAt   DateTime?
  publishedAt  DateTime?
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt

  artist       ArtistProfile @relation(fields: [artistId], references: [id], onDelete: Cascade)
  blocks       Block[]
  tracks       Track[]
  memberships  Membership[]
  posts        Post[]
  preSaveLinks PreSaveLink[]
  events       AnalyticsEvent[]

  @@unique([artistId, slug])
  @@index([status, releaseDate])
}

model Block {
  id        String   @id @default(cuid())
  partyId   String
  type      String   // BlockType — validated by zod, not enum, so configs stay flexible
  order     Int
  config    Json     // shape per type: docs/05-blocks-spec.md
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  party     Party      @relation(fields: [partyId], references: [id], onDelete: Cascade)
  pollVotes PollVote[]
  answers   QuestionAnswer[]

  @@index([partyId, order])
}

// ─────────────────────────────── Audio ─────────────────────────────────────

enum PlayRule {
  UNLIMITED
  LIMITED   // playLimit N plays per user
}

model Track {
  id           String   @id @default(cuid())
  partyId      String
  title        String
  fileKey      String   // R2 key — NEVER sent to the client
  mimeType     String
  durationSec  Int?
  playRule     PlayRule @default(UNLIMITED)
  playLimit    Int?     // required when playRule=LIMITED (1,2,3,…)
  availableFrom DateTime? // date-limited window
  expiresAt    DateTime? // after this, playback blocked (usually release date)
  createdAt    DateTime @default(now())

  party Party  @relation(fields: [partyId], references: [id], onDelete: Cascade)
  plays Play[]
}

model Play {
  id               String    @id @default(cuid())
  trackId          String
  userId           String
  startedAt        DateTime  @default(now())
  completedPct     Int       @default(0) // 0–100, updated by heartbeat
  deviceFingerprint String?
  track            Track     @relation(fields: [trackId], references: [id], onDelete: Cascade)
  user             User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([trackId, userId])
}

// ───────────────────────────── Membership ──────────────────────────────────

model Membership {
  id                  String    @id @default(cuid())
  userId              String
  partyId             String
  joinedAt            DateTime  @default(now())
  referredByCode      String?   // referral code used at join
  referralCode        String    @unique // this member's own share code (nanoid 8)
  completedExperience Boolean   @default(false)
  preSavedAt          DateTime?
  emailOptOut         Boolean   @default(false)
  furthestAct         Int       @default(0)  // highest act index reached (guided-first-pass unlock state)
  lastSeenAt          DateTime? // touched on experience fetch + chat polls; presence = lastSeenAt > now-5min

  user  User  @relation(fields: [userId], references: [id], onDelete: Cascade)
  party Party @relation(fields: [partyId], references: [id], onDelete: Cascade)

  @@unique([userId, partyId])
  @@index([partyId])
  @@index([partyId, lastSeenAt])
}

model PreSaveLink {
  id       String @id @default(cuid())
  partyId  String
  platform String // spotify | apple | amazon | youtube | deezer
  url      String
  party    Party  @relation(fields: [partyId], references: [id], onDelete: Cascade)

  @@unique([partyId, platform])
}

// ───────────────────────────── Community ───────────────────────────────────
// IMPORTANT: Room Chat and the Community feed are the SAME Post stream, rendered
// two ways (docs/04). The chat rail shows posts chronologically ascending (no media,
// hearts = likes); the community page shows them pinned-first descending with
// comments. One model, no separate ChatMessage table.

model Post {
  id        String    @id @default(cuid())
  partyId   String
  authorId  String
  body      String    @db.Text
  mediaKey  String?   // R2 key (image only in MVP)
  pinned    Boolean   @default(false)
  createdAt DateTime  @default(now())
  deletedAt DateTime?

  party    Party     @relation(fields: [partyId], references: [id], onDelete: Cascade)
  author   User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  comments Comment[]
  likes    Like[]
  reports  Report[]

  @@index([partyId, pinned, createdAt])
}

model Comment {
  id        String    @id @default(cuid())
  postId    String
  authorId  String
  parentId  String?   // one level of nesting only (replies to replies attach to same parent)
  body      String    @db.Text
  createdAt DateTime  @default(now())
  deletedAt DateTime?

  post    Post      @relation(fields: [postId], references: [id], onDelete: Cascade)
  author  User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  parent  Comment?  @relation("replies", fields: [parentId], references: [id], onDelete: Cascade)
  replies Comment[] @relation("replies")
  likes   Like[]

  @@index([postId, createdAt])
}

model Like {
  id        String   @id @default(cuid())
  userId    String
  postId    String?
  commentId String?
  createdAt DateTime @default(now())

  user    User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  post    Post?    @relation(fields: [postId], references: [id], onDelete: Cascade)
  comment Comment? @relation(fields: [commentId], references: [id], onDelete: Cascade)

  @@unique([userId, postId])
  @@unique([userId, commentId])
}

// ──────────────────────── Polls & Questions (blocks) ───────────────────────

model PollVote {
  id        String   @id @default(cuid())
  blockId   String
  userId    String
  optionIdx Int
  createdAt DateTime @default(now())
  block     Block    @relation(fields: [blockId], references: [id], onDelete: Cascade)
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([blockId, userId])
}

model QuestionAnswer {
  id        String   @id @default(cuid())
  blockId   String
  userId    String
  answer    String   @db.Text
  createdAt DateTime @default(now())
  block     Block    @relation(fields: [blockId], references: [id], onDelete: Cascade)
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([blockId, userId])
}

// ─────────────────────── Notifications & analytics ─────────────────────────

enum NotificationType {
  PARTY_LIVE
  ARTIST_POSTED
  SONG_RELEASED
  NEW_PARTY
  COMMUNITY_REPLY
}

model Notification {
  id        String           @id @default(cuid())
  userId    String
  type      NotificationType
  title     String
  body      String?
  url       String           // in-app destination
  readAt    DateTime?
  createdAt DateTime         @default(now())
  user      User             @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId, readAt, createdAt])
}

model AnalyticsEvent {
  id        String   @id @default(cuid())
  partyId   String?
  userId    String?
  type      String   // taxonomy: docs/08-notifications-analytics.md
  meta      Json     @default("{}")
  country   String?  // from x-vercel-ip-country / dev fallback null
  device    String?  // mobile | desktop | tablet (ua parse)
  referrer  String?
  createdAt DateTime @default(now())

  party Party? @relation(fields: [partyId], references: [id], onDelete: SetNull)

  @@index([partyId, type, createdAt])
}

// ─────────────────────────────── Admin ─────────────────────────────────────

model Report {
  id         String    @id @default(cuid())
  reporterId String
  postId     String?
  reason     String
  resolvedAt DateTime?
  createdAt  DateTime  @default(now())
  reporter   User      @relation("reporter", fields: [reporterId], references: [id], onDelete: Cascade)
  post       Post?     @relation(fields: [postId], references: [id], onDelete: Cascade)
}

model FeatureFlag {
  key       String  @id
  enabled   Boolean @default(false)
  note      String?
}
```

## Notes for the builder

- `Block.type` is a `String`, not a Prisma enum, so adding block types never needs a
  migration; validity is enforced by the zod discriminated union in `types/blocks.ts`.
- Play-limit accounting counts **rows in `Play`** per (trackId, userId). A play row is
  created when a play token is redeemed (see 07), not when the button is clicked.
- `Membership.referralCode` is generated with `nanoid(8)` at join time.
- Soft delete (`deletedAt`) only on Post/Comment; render as "[removed]" placeholder if it has replies, hide otherwise.
- Seed (`prisma/seed.ts`) must create: admin user (`admin@listeningparty.local`),
  artist user Victor (`victor@listeningparty.local`, handle `victor`, profile filled),
  one PUBLISHED party `new-single` with 8+ blocks covering every block type at least
  once across seed data, one track (generate a 10-second sine-wave MP3 with ffmpeg if
  available, else commit a tiny silent MP3 fixture into `prisma/fixtures/`), playRule
  LIMITED/2, three fan users with memberships, a few posts/comments/likes, pre-save
  links for spotify + apple, and ~50 randomized AnalyticsEvents so the analytics page
  renders non-empty charts.
```
