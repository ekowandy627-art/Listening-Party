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

The repo already exists at `https://github.com/ekowandy627-art/Listening-Party` with
this spec pack pushed to `main`. Point the build session at that repo (clone it, or
open it if already local) and give it this prompt:

> Build the Listening Party MVP exactly as specified in this repo's `docs/` folder —
> read `README.md` first for the doc index. Follow `docs/09-build-plan.md` milestone
> by milestone, starting at M0. Do not re-litigate stack, schema, or UI decisions —
> they are final in `docs/00-decisions.md`, `docs/02-data-model.md`, and
> `docs/12-theming.md`. Scaffold the Next.js app into `app/` inside this existing repo
> (do not `git init` fresh) and commit after every milestone. After each milestone, run
> the checks listed for it before moving on.
>
> M0–M11 need no external accounts — everything runs locally against Docker
> (Postgres + MinIO); magic links print to the console. Build and verify all of these
> against `docs/10-acceptance-checklist.md` (everything except the final §Deployment
> section). **Stop after M11 and report status** rather than proceeding to M12 — I
> don't have Vercel/Neon/Cloudflare/Resend accounts set up yet, so deployment
> (`docs/11-deployment.md`) is a separate step we'll do once those exist.

Use this **shorter follow-up prompt** later, once the four accounts exist, to finish
the job:

> Continue the Listening Party build at M12 per `docs/11-deployment.md` — deploy to
> production (Vercel + Neon + R2 + Resend, no custom domain, no Google/Apple OAuth),
> run the production seed for artist "Challe from Ghana" (handle `challefromghana`),
> and verify the §Deployment section of `docs/10-acceptance-checklist.md` on the live
> URL.

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
| [docs/05-blocks-spec.md](docs/05-blocks-spec.md) | The 15 block types ("acts"): config schemas, editor UI, fan-side rendering |
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
