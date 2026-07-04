# Listening Party

## Product Requirements Document (PRD)

Version: MVP 1.0

---

# 1. Overview

Listening Party is a platform that helps artists transform music releases into engaging experiences that convert casual listeners into lifelong fans.

Instead of simply sharing a pre-save link or hosting a one-time livestream, artists create immersive release experiences where fans:

* Discover the story behind the music
* Listen to exclusive pre-release songs securely
* Interact with the artist
* Join the artist's community
* Automatically receive future release notifications
* Pre-save new releases with minimal effort

The platform removes the friction involved in building fan communities while maximizing release-day engagement.

---

# 2. Vision

To become the world's leading platform for artist-first music releases by making every song release an interactive experience rather than a single event.

Listening Party should become what Substack is for writers and Patreon is for creators.

When fans think of an artist's community, they should think of Listening Party.

---

# 3. Problem Statement

Today's music release journey is fragmented.

Artists use: Instagram, TikTok, Discord, WhatsApp, Linktree, Feature.fm, Spotify Pre-save, Email newsletters, Livestream platforms.

Each solves one problem. None combine: Community, Storytelling, Exclusive listening, Marketing, Analytics, Conversion.

Artists spend more time managing tools than engaging fans.

---

# 4. Goals

## Artist Goals

* Increase pre-saves
* Increase release-day streams
* Build an owned audience
* Grow fan engagement
* Understand fan behaviour
* Share the inspiration behind music
* Create excitement before release
* Sell merchandise later

## Fan Goals

* Hear music before everyone else
* Learn the story behind songs
* Feel closer to the artist
* Interact with other fans
* Receive exclusive content
* Never miss future releases

---

# 5. Target Users

## Primary

Independent Artists, Emerging Artists, Bands, Music Labels, Managers, Producers

## Secondary

Podcasters, Authors, Filmmakers, Content Creators — anyone launching creative work.

---

# 6. Core Philosophy

A release should not be: Song → Released → Forgotten

Instead: Story → Journey → Listening Experience → Community → Release → Long-term Relationship

---

# 7. User Types

### Artist

Creates listening parties, uploads songs, publishes content, views analytics, builds community.

### Fan

Joins listening parties, listens, comments, pre-saves, shares, follows artists.

---

# 8. User Journey

## Artist

Create Listening Party → Upload artwork → Upload welcome video → Upload story → Upload exclusive song → Configure listening rules → Add pre-save → Publish → Invite fans → Release song → Continue community engagement

## Fan

Receive invite → Enter email → Receive Magic Link → Enter Listening Party → Watch welcome → Watch story → Listen → Comment → Pre-save → Join community → Receive future updates

---

# 9. MVP Functional Requirements

## Module 1 — Authentication

Objective: reduce signup friction.

* User joins through a shareable link: `listeningparty.app/victor/new-single`
* Fan clicks. Screen displays: "Join Victor's Listening Party" / "Enter your email" / Continue
* After entering email: system sends Magic Link. User clicks Magic Link. System creates account automatically.
* Email becomes unique account identifier. Password is optional. Users can later create passwords.
* Support: Magic Link, Google Login, Apple Login. Future: Spotify Login.

## Module 2 — Artist Dashboard

Artist can: Create Listening Party, Edit, Duplicate, Delete, Archive.

Dashboard fields: Title, Description, Cover Art, Banner, Release Date, Genre, Status (Draft / Published / Live / Completed).

## Module 3 — Experience Builder

Artist creates a sequence. Each section is called a Block.

Supported Blocks: Welcome Video, Video Story, Photo Gallery, Lyrics, Voice Note, Text, Countdown, Poll, Question, Audio Player, Artist Message, Merch Promotion, Pre-save CTA, Community Invitation.

Artist chooses order. Drag and Drop.

## Module 4 — Secure Audio

Requirements: Encrypted streaming, No download, No right click, No URL exposure, Play limit, Optional watermark, Device fingerprint, Expiry date, Screen recording detection where supported.

Playback Rules: Unlimited, One Play, Two Plays, Three Plays, Date Limited, Time Limited.

Example: "This song may only be played twice before release."

## Module 5 — Community

Each Listening Party creates a community.

Features: Posts, Comments, Replies, Likes, Pinned announcements, Artist posts, Fan posts, Media uploads, Search, Notifications.

Communities remain active after release.

## Module 6 — Pre-save

Supported: Spotify, Apple Music, Amazon Music, YouTube Music, Deezer.

Artist pastes one pre-save link. Platform displays: Pre-save / Done / Reminder Set.

### Future: Auto Pre-save

Fan clicks "Connect Spotify". Spotify requests permission. If approved, future releases automatically pre-save. Fan can disconnect anytime.

Important: Email alone cannot authorize future pre-saves. Users must explicitly grant permission through Spotify or Apple Music. This complies with the platforms' authentication and privacy requirements.

## Module 7 — Notifications

Fans receive: Party Live, Artist Posted, Song Released, New Listening Party, Merch Launch, Community Reply.

Supported: Email, Push. SMS (Future), WhatsApp (Future).

## Module 8 — Analytics

Artist Dashboard: Invited, Joined, Completed Experience, Average Watch Time, Listening Completion, Most Replayed Songs, Community Members, Comments, Shares, Pre-save Rate, Conversion Rate, Countries, Cities, Devices, Referral Sources.

## Module 9 — Sharing

Every Listening Party has: Unique Link, QR Code, Referral Link. Users earn referral badges. Future: Rewards, Exclusive content.

## Module 10 — Artist Profile

Artist: Photo, Bio, Genres, Links, Upcoming Releases, Past Listening Parties, Community Feed, Merch, Events. Fans follow artists.

## Module 11 — Search

Search by: Artist, Genre, Song, Community. Trending. Featured.

## Module 12 — Admin

Manage: Users, Artists, Reports, Communities, Songs, Analytics, Feature Flags, Moderation.

---

# 10. Non-functional Requirements

Responsive Web, Mobile First, Cloud Hosted, CDN Streaming, 99.9% uptime, Scalable, Secure, Encrypted storage, GDPR compliant, Accessibility (WCAG AA), Fast page loads (<2s target).

---

# 11. Future Roadmap

## Phase 2

Live Listening Rooms, Voice Chat, Live Video, Live Reactions, Artist Q&A, Listening Countdown, Scheduled Premieres.

## Phase 3

Merchandise Store, Ticket Sales, VIP Fan Clubs, Subscriptions, Donations, Digital Collectibles, Meet & Greet Booking, Exclusive Releases.

## Phase 4

Creator Economy Platform, Creator CRM, Fan Segmentation, Email Campaigns, Tour Management, Music Distribution, Label Tools, Brand Partnerships, Creator Marketplace, AI Fan Insights, AI Release Recommendations.

---

# 12. Technical Architecture (MVP)

**Frontend:** React (Next.js), Tailwind CSS, Progressive Web App (PWA)

**Backend:** NestJS or Node.js, REST API (GraphQL optional in later phases)

**Database:** PostgreSQL

**Storage:** Cloudflare R2 or AWS S3

**Streaming:** Encrypted HLS, Signed URLs with expiry, Optional forensic audio watermarking for premium plans

**Authentication:** Magic Link (recommended for MVP), Google OAuth, Apple Sign-In

**Notifications:** Email (Resend/Postmark), Push Notifications (later), WhatsApp integration (later)

**Payments (Future):** Stripe

**Hosting:** Vercel (frontend), Railway/Fly.io/AWS (backend)

---

# 13. What Makes Listening Party Different

It combines four capabilities into a single experience:

1. **Storytelling** — Artists control the narrative around every release.
2. **Secure Early Access** — Fans experience unreleased music in a protected environment with configurable listening limits.
3. **Community** — Every release builds a persistent community rather than a one-time audience.
4. **Conversion** — Pre-saves, referrals, notifications, and future releases are all managed from one place.

The long-term vision is to make Listening Party the home where artists release music, grow communities, and build lasting relationships with their fans, rather than depending on fragmented social platforms.
