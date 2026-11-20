# GHOST — Build Plan

Personal data-exposure discovery, removal, and monitoring.
Assumed path: **family first (≤10 consenting users), then commercial.** Architecture is built multi-tenant from day one so the family version becomes the beta without a rewrite.

## Guiding constraints

- Only act for verified, consenting people (or their legal guardian). No "look up anyone" feature.
- The vault holding user PII is the highest-value target in the system — encrypt per field, minimize, delete on request.
- AI drafts and classifies; deterministic code decides what data goes where. The LLM never holds raw SSNs/DOBs and never picks a destination URL.
- No dark-web crawling. Use licensed breach feeds only.
- Report removals only when re-verified, never "request sent = removed."

## Stack

| Layer | Choice | Why |
|---|---|---|
| Web app | Next.js (App Router) + TypeScript, Tailwind | One codebase, fast to ship |
| Auth | Passkeys + TOTP (e.g. Clerk or Auth.js) | Phishing-resistant; it's a privacy product |
| API / workers | Python (FastAPI) | Best scraping/automation ecosystem |
| Job orchestration | Temporal (or Postgres-backed queue like Procrastinate for MVP) | Removals run for weeks with retries and timers |
| Browser automation | Playwright, residential-quality egress only where ToS allows | Opt-out forms |
| DB | Postgres; PII columns encrypted with envelope keys (cloud KMS) | Auditable, simple |
| LLM | Claude API (`claude-sonnet-5-5` for workflows, `claude-haiku-5-5` for classification) | Form understanding, match scoring, email parsing |
| Email | Dedicated per-user relay inbox (e.g. `u123@mail.ghost.app`) via Postmark/SES inbound | Catches broker confirmation links |
| Breach data | Have I Been Pwned API (paid key) to start; evaluate SpyCloud/Constella later | Licensed, legal |
| Hosting | Vercel (web) + Fly.io/Render (workers) → AWS later for compliance | Cheap to start |

## Data model (core tables)

- `subjects` — person being protected; encrypted identifiers (names/aliases, emails, phones, addresses, DOB year).
- `authorizations` — who may act for a subject, signed consent, scope, expiry.
- `brokers` — site registry: removal method (form / email / API / DROP-covered), required fields, ToS notes, avg turnaround.
- `exposures` — subject × source × URL, matched fields, confidence score, screenshot hash.
- `removal_requests` — state machine: `drafted → awaiting_approval → submitted → awaiting_confirmation → confirmed → verified_removed | failed | reappeared`.
- `evidence` — screenshots, emails, timestamps (immutable, hash-chained audit log).
- `breach_hits` — source breach, exposed data classes, recommended actions.

## Phase 0 — Groundwork (week 1)

- Repo, CI, IaC, secrets in KMS, encrypted-field library with tests.
- Broker registry seeded from the **California Data Broker Registry** (public CSV) + top ~30 people-search sites by traffic.
- Write a one-page threat model and data-retention policy before storing any real PII.

## Phase 1 — MVP for family (weeks 2–5)

Goal: measurably remove real listings for 2–3 people within 30 days.

1. **Onboarding** — passkey signup, subject profile, consent capture, ID check (Stripe Identity / Persona).
2. **Discovery v1** — per-site search adapters for 20–30 people-search sites; fetch result pages, extract candidate records, score match against profile (deterministic field overlap + Haiku tiebreak). Low-confidence matches go to the user to confirm — never auto-remove an uncertain match.
3. **Removal v1** — three handlers:
   - Email/letter requests (CCPA/state-law templates, deadline tracking — 45 days for CCPA).
   - Scripted Playwright flows for the top ~10 sites (hand-written, tested).
   - "Guided manual" checklist for CAPTCHA/ID-upload sites.
4. **DROP integration** — walk California residents through the state DROP request (free); mark the ~600 covered brokers so we don't duplicate effort.
5. **Dashboard** — exposures, request status, verified-removal count, evidence per item.
6. **Breach check** — HIBP lookup for each email; plain-language action list (change password, enable 2FA, freeze credit).

Exit criteria: ≥60% of discovered listings verified removed at day 30 on the supported sites.

## Phase 2 — Automation & monitoring (weeks 6–9)

- **AI removal agent** for long-tail sites: Claude reads the opt-out page, produces a structured plan (fields needed, steps); a deterministic executor runs it only if every field is on the site's allowlist and the domain matches the registry. Unknown plans → human review queue.
- Inbound email parser: auto-click verified confirmation links (domain-matched only), classify broker replies.
- Re-scan scheduler: every 30 days per site; reappearances auto-reopen a request.
- Google: guide users through "Results about you" (no public API) and track outcomes.
- Exposure score with evidence (weighted: home address, phone, relatives > email > name only).
- Family accounts: one owner, multiple subjects, per-subject consent.

## Phase 3 — Commercial beta (weeks 10–14+)

- Stripe billing (individual / family tiers), self-serve account + data deletion.
- Expand to 100+ adapters; add adapter health monitoring (sites change forms constantly — this is the real ongoing cost).
- Security: external pen test, SOC 2 readiness tooling (Vanta/Drata), incident-response runbook.
- Legal review: authorized-agent rules per state, ToS posture for automation, marketing claims (no "erase you from the internet").
- Support tooling and an ops console for the manual-review queue.

## Key risks

| Risk | Mitigation |
|---|---|
| Vault breach | Field-level encryption, short retention, no PII in logs or LLM prompts beyond what a form needs |
| Wrong-person removal / matching errors | Confidence threshold + user confirmation |
| Abuse (stalker uses tool to find someone) | Discovery only for verified self/dependents; no free-text search of others |
| Sites block automation / CAPTCHAs | Prefer email/legal requests; guided manual fallback |
| Adapter rot | Nightly synthetic tests per adapter, alerting |
| Competing with free DROP | Position as discovery + non-CA coverage + verification + monitoring |

## Rough cost (family MVP)

Hosting ~$50–100/mo, HIBP ~$4–40/mo, Claude API ~$20–50/mo, ID verification ~$1–2/check, email ~$15/mo.

## First tasks

1. Scaffold monorepo (`apps/web`, `services/api`, `services/workers`, `packages/crypto`).
2. Encrypted-field library + Postgres schema + migrations.
3. Broker registry import + first 5 people-search adapters (discovery only).
4. Onboarding + consent flow.
5. Manual end-to-end run for one subject; measure.
