# Osintgram Web

**An open-source OSINT platform for authorized investigators.**

[![Status: pre-alpha](https://img.shields.io/badge/status-pre--alpha-orange)](docs/superpowers/plans/2026-08-11-osintgram-web-implementation.md)
[![License: GPLv3](https://img.shields.io/badge/license-GPLv3-blue)](#license-and-attribution)
[![Python 3.14](https://img.shields.io/badge/python-3.14-blue)](backend/pyproject.toml)
[![FastAPI](https://img.shields.io/badge/backend-FastAPI-009688)](https://fastapi.tiangolo.com)
[![React 19](https://img.shields.io/badge/frontend-React%2019-61dafb)](https://react.dev)

Osintgram Web turns the [Osintgram](https://github.com/Datalux/Osintgram) command-line
tool into a hosted, multi-user web application: a browser-based workspace for collecting
and analysing **publicly available** Instagram information during lawful investigations.

It is built for teams that today run OSINT collection from a terminal on one analyst's
laptop, with no access control, no record of who searched for whom, and no way to review
or hand over the results.

---

## ⚠️ Project status: pre-alpha — not yet functional

**Nothing in this repository works yet.** It currently contains the upstream
[Full Stack FastAPI Template](https://github.com/fastapi/full-stack-fastapi-template)
plus a written specification and implementation plan. No Osintgram functionality has
been implemented.

| Artefact | Status |
|---|---|
| [Requirements & design specification](docs/superpowers/specs/2026-08-11-osintgram-web-design.md) | Complete |
| [Implementation plan](docs/superpowers/plans/2026-08-11-osintgram-web-implementation.md) | Phase 1 detailed, phases 2–8 scoped |
| Phase 1 — data layer | Not started |
| Phases 2–8 | Not started |

Do not deploy this. Do not rely on it for casework.

---

## Who this is for

Investigators who already have a lawful basis to research a subject and need to do it
in a way that is repeatable, attributable, and reviewable:

- **Law enforcement and public agencies** conducting authorized investigations
- **Licensed private investigators** working a documented engagement
- **Trust & safety and fraud teams** investigating platform abuse
- **Journalists and academic researchers** working within their institution's ethics framework

Being open source, this software cannot verify who runs it. Nothing here grants
authority, and installing it does not make a search lawful. **Establishing lawful
basis is entirely the operator's responsibility.**

---

## What it does and does not do

Being precise about this matters, because tools in this space are routinely
misrepresented.

### It can

- Retrieve profile details of a **public** Instagram account
- List followers and accounts followed
- Collect post metadata: captions, hashtags, like and comment counts, media types
- Identify accounts that tagged or commented on the subject's posts
- Download publicly visible photos, stories and profile pictures
- Surface contact details and post locations **that the account holder chose to publish**

### It cannot

- **Access private profiles.** If an account is private, its content is unavailable.
  Any tool claiming otherwise is misrepresenting itself.
- **Access direct messages, deleted content, or anything requiring a warrant.** Nothing
  here compels disclosure — for that you need legal process served on the platform.
- **Log in as, impersonate, or interact with any account.** It is read-only and never
  posts, follows, or messages.
- **Bypass any access control, rate limit, or authentication mechanism.**
- **Produce court-ready evidence on its own.** See below.

### Not a digital forensics tool

This platform records *who ran what, against whom, and when* for accountability. It does
**not** currently provide chain of custody, cryptographic hashing of collected artefacts,
tamper-evident logging, or timestamping to an evidentiary standard.

If collected material may become evidence, capture it through your organisation's
approved forensic process. Treat output here as investigative lead material, not exhibits.

---

## Lawful and responsible use

By operating this software you accept responsibility for the following.

1. **Lawful basis.** Have documented authority for each subject you research. "It is
   public" is not by itself a lawful basis for systematic collection in most jurisdictions.
2. **Data protection.** Collection of personal data is regulated — India's DPDP Act 2023,
   the EU/UK GDPR, and comparable regimes elsewhere. Purpose limitation, minimisation and
   retention limits apply to investigators too.
3. **Third parties.** Follower and contact commands return personal data about people who
   are **not** your subject and have no connection to your investigation. Collect the
   minimum necessary.
4. **Platform terms.** Automated collection may conflict with Instagram's Terms of Service.
   Assess that against your legal authority.
5. **No harassment or targeting.** This must not be used to stalk, dox, intimidate, or
   surveil individuals without authority, nor to build profiles of people based on
   protected characteristics.

To make these enforceable rather than aspirational, the design gates the most sensitive
commands behind per-account approval and an immutable audit log
([spec §11](docs/superpowers/specs/2026-08-11-osintgram-web-design.md)).

**The maintainers accept no responsibility for how this software is used.**

---

## Planned capabilities

20 collection commands, in three access tiers.

**Standard — available to any active account**

| Command | Returns |
|---|---|
| `info` | Subject profile details |
| `followers` / `followings` | Accounts following, or followed by, the subject |
| `captions` | Post captions |
| `comments` / `commentdata` | Comment totals, and full comment threads |
| `likes` | Like totals across posts |
| `hashtags` | Hashtags the subject uses |
| `mediatype` | Breakdown of photo versus video posts |
| `tagged` | Accounts the subject tagged |
| `wtagged` / `wcommented` | Accounts that tagged, or commented on, the subject |

**Media — writes files to object storage**

| Command | Returns |
|---|---|
| `propic` | Profile picture |
| `photos` | Publicly visible photos |
| `stories` | Currently visible stories |

**Restricted — approved accounts only, every use audited**

| Command | Returns |
|---|---|
| `addrs` | Locations attached to the subject's posts |
| `fwersemail` / `fwingsemail` | Published email addresses of followers / followed accounts |
| `fwersnumber` / `fwingsnumber` | Published phone numbers of followers / followed accounts |

The restricted tier returns personal data about **third parties in bulk**. Access is
granted per account by an administrator, and every query is recorded permanently.

---

## Architecture

```
Browser ── Traefik ── FastAPI ── Postgres      (users, jobs, audit)
                         │
                         ├──── Redis          (queue, quotas, rate limits)
                         │
                     Celery worker ── HikerAPI (the only component making upstream calls)
                         │
                         └──── S3-compatible storage (collected media, auto-expiring)
```

Two properties drive the design:

- **The API never makes an upstream call during a request.** All collection happens in
  workers, which keeps requests fast and puts cost accounting behind a single choke point.
- **Collection is asynchronous because it has to be.** Contact-detail commands issue one
  upstream request *per follower* — a 10,000-follower subject means 10,000 requests. Jobs
  run in the background with live progress, and their projected cost is shown for
  confirmation before they start.

**Stack:** Python 3.14, FastAPI, SQLModel, Alembic, PostgreSQL 18, Celery, Redis ·
React 19, TypeScript, Vite, Tailwind CSS, shadcn/ui, TanStack Router/Query ·
Docker Compose, Traefik, Playwright, GitHub Actions.

### Data access

Instagram data is retrieved through [HikerAPI](https://hikerapi.com), a third-party
service. Instagram account passwords are **never** accepted, stored, or transmitted by
this application — the legacy password-based backend of the original CLI is deliberately
excluded.

Each user supplies their own HikerAPI token, encrypted at rest, so collection costs and
quotas are attributed to whoever incurred them.

---

## Getting started

Nothing is implemented yet, so there is no application to run. To work on it:

```bash
git clone git@github.com:Dadiya-Harsh/Osintgram-full-stack-app.git
cd Osintgram-full-stack-app
cp .env.example .env    # then replace every "changethis" value
docker compose watch
```

See [development.md](development.md) for the local workflow and
[deployment.md](deployment.md) for deployment. Start implementation at
[Phase 1 of the plan](docs/superpowers/plans/2026-08-11-osintgram-web-implementation.md).

> **Never commit `.env`.** It is git-ignored; `.env.example` is the tracked template.

---

## Contributing

Contributions are welcome. Please read the
[specification](docs/superpowers/specs/2026-08-11-osintgram-web-design.md) first — it
records decisions and their reasoning, and pull requests that contradict it without
discussion are hard to evaluate.

Non-negotiable constraints, enforced by CI:

- No `print()` in application code — ruff `T201` is enabled
- `mypy` runs in strict mode
- TLS certificate verification is never disabled
- Secrets never appear in responses, logs, or error messages
- Tests never call the live HikerAPI; all upstream behaviour comes from fixtures

Run `bash backend/scripts/lint.sh` and `uv run pytest` before opening a pull request.

### Security disclosures

Please report vulnerabilities privately through GitHub's security advisory feature rather
than opening a public issue.

---

## License and attribution

This project derives from two upstream works:

| Project | Author | License |
|---|---|---|
| [Osintgram](https://github.com/Datalux/Osintgram) | Giuseppe Criscione | **GPLv3** |
| [Full Stack FastAPI Template](https://github.com/fastapi/full-stack-fastapi-template) | Sebastián Ramírez | MIT |

Because Osintgram is GPLv3 and this project incorporates its logic, **the combined work
must be distributed under GPLv3**. MIT is compatible with that direction, so the template's
contribution is preserved under GPLv3 terms with attribution retained.

[`LICENSE`](LICENSE) contains the full GPLv3 text. [`NOTICE`](NOTICE) records third-party
attribution, including the template's original MIT license, which the MIT terms require
to be preserved.

> **Note for hosted deployments:** GPLv3 obligations are triggered by *distribution*.
> Running this as a network service is generally not distribution under GPLv3 — that is
> what the AGPL was written to cover — so users of a hosted instance would have no
> automatic right to the source. If you intend them to, license the deployment under
> AGPLv3 instead. Worth confirming with counsel before deploying for an agency.
