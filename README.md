# Listening Party — One-Take Build Pack

This folder contains everything needed to build the Listening Party MVP in a single
session with no open questions. Read the docs in order; every decision has already
been made.

## What Listening Party is

A platform where artists turn music releases into immersive experiences: fans join
via a shareable link + magic-link email auth, walk through a story-driven sequence
of "blocks" (video, lyrics, photos, polls…), listen to unreleased tracks under
play-limit rules, pre-save, and join a persistent community that survives the release.

## The one-take build prompt

When starting the build session, use this prompt:

> Build the Listening Party MVP exactly as specified in `listening-party/docs/`.
> Follow `docs/09-build-plan.md` milestone by milestone (M0–M12). Do not re-litigate
> stack or schema decisions — they are final in `docs/00-decisions.md` and
> `docs/02-data-model.md`. After each milestone, run the checks listed for it.
> After the local acceptance pass (M11), deploy to production per
> `docs/11-deployment.md` — ask me for Vercel/Neon/Cloudflare/Resend logins at M12,
> not earlier. Done means the full `docs/10-acceptance-checklist.md` is green,
> including the §Deployment section on the live URL.

## Document index

| Doc | Contents |
|---|---|
| [PRD.md](PRD.md) | Original product requirements document (source of truth for scope) |
| [docs/00-decisions.md](docs/00-decisions.md) | Final tech stack + every deviation from the PRD, with reasons |
| [docs/01-architecture.md](docs/01-architecture.md) | System architecture, project layout, env vars, local dev setup |
| [docs/02-data-model.md](docs/02-data-model.md) | Complete Prisma schema — copy verbatim |
| [docs/03-api-spec.md](docs/03-api-spec.md) | Every API route: method, path, auth, request/response shapes |
| [docs/04-routes-and-screens.md](docs/04-routes-and-screens.md) | Every page: URL, layout, components, states, empty states |
| [docs/12-theming.md](docs/12-theming.md) | **Visual system (source of truth for UI):** logo, palette, type, 6 room mood-presets, component styling from the approved mockups |
| [docs/05-blocks-spec.md](docs/05-blocks-spec.md) | The 14 block types: config schemas, editor UI, fan-side rendering |
| [docs/06-auth-spec.md](docs/06-auth-spec.md) | Magic link + Google/Apple flows, session model, role gating |
| [docs/07-secure-audio-spec.md](docs/07-secure-audio-spec.md) | Upload pipeline, proxied streaming, play-limit enforcement |
| [docs/08-notifications-analytics.md](docs/08-notifications-analytics.md) | Email notification triggers + analytics event taxonomy |
| [docs/09-build-plan.md](docs/09-build-plan.md) | Ordered milestones M0–M12 with per-milestone verification |
| [docs/10-acceptance-checklist.md](docs/10-acceptance-checklist.md) | Final testable acceptance criteria per module, incl. production smoke test |
| [docs/11-deployment.md](docs/11-deployment.md) | Production deployment: Vercel + Neon + R2 + Resend, prod seed, smoke test |

**Brand assets:** drop the logo files into `app/public/brand/` (`lp-full.svg`,
`lp-mono-black.svg`, `lp-mono-white.svg`, `lp-icon.svg`) before/at M0 — see docs/12.

## Scope guardrails

- **In scope (MVP):** everything in PRD sections 9 (Modules 1–12) as narrowed by
  `docs/00-decisions.md`.
- **Out of scope:** everything in PRD section 11 (Phases 2–4), auto pre-save via
  Spotify OAuth, SMS/WhatsApp/push notifications, payments, forensic watermarking,
  screen-recording detection, native apps.
- When the PRD and a doc in `docs/` conflict, **`docs/` wins** — it is the
  fleshed-out, buildable interpretation.
