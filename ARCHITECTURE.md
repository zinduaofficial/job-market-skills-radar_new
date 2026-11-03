# Architecture: from monthly crawl to two frontends

**Status:** proposal. It assumes the merge in [MERGE_PLAN.md](MERGE_PLAN.md), because the monthly run and D1 export only exist in `job_skills_radar` today.
**Date:** 3 November 2026

## Summary

Once a month, a GitHub Actions workflow runs the radar, loads a snapshot into a Cloudflare D1 database, and rebuilds two frontends:

- **External** (prospects and customers, in `website-monorepo/apps/job-radar`): static HTML built from D1 at deploy time.
- **Internal** (Zindua staff, in this repo): server-rendered pages that query D1 live, behind Cloudflare Access.

```
GitHub Actions — .github/workflows/monthly.yml, 03:00 UTC on the 2nd
  │
  ├─ restore data/raw/jobs.sqlite from R2
  ├─ jobradar monthly        collect → extract → export → health checks
  ├─ save jobs.sqlite to R2 (+ dated backup)
  ├─ commit exports to git   data/processed, site, d1, history, runs
  │
  ├─ warnings? ── yes → open an issue, publish nothing, last month stays live
  │
  ├─ jobradar publish --d1   snapshot loads as 'loading', flips to 'published'
  └─ Pages deploy hooks
        ├─ External: build reads D1 → static HTML on the CDN
        └─ Internal: SSR, queries D1 per request, behind Cloudflare Access
```

## Components

| Part | Where | Job |
| --- | --- | --- |
| Collector | GitHub Actions in this repo | Runs `jobradar monthly` on a schedule, or by hand |
| Raw store | R2 bucket `job-radar`: `raw/jobs.sqlite`, `backups/` | The only copy of every posting's full text |
| Published data | D1 database `job-radar` | One snapshot per month; readers use the `current_snapshot` view |
| Exports in git | `data/` in this repo | Processed CSVs, site JSON, D1 SQL, history and run reports, for review and diffing |
| External frontend | `website-monorepo/apps/job-radar` | Public pages for prospects, rebuilt after each publish |
| Internal frontend | This repo, e.g. `web/` | Staff views with filters and drill-downs |

## Decisions and why

**The crawl runs in GitHub Actions, not a Cloudflare Worker or a server.**
The pipeline is Python, obeys rate limits as low as 0.4 requests a second, and needs a SQLite store of about 46,000 postings. A Worker can't run that. Actions minutes are free and unlimited for this public repo, and a job may run for 6 hours. The monthly run fetches only postings it doesn't have yet. It excludes the one-off Wayback backfill (`monthly: false`). Move to a self-hosted runner only if a job board blocks GitHub's IP addresses or a run nears 6 hours.

**The raw store lives in R2.**
It is not in git (too large, and rebuildable data only), and it can't live on one laptop: a sync client corrupted it once. The workflow restores it before the run and saves it after, even if a later step fails. Restoring fails loudly if the file is missing. Starting empty would re-crawl from scratch and lose the Kenyan history that only Wayback and Career Point provide. The HTTP cache is not kept between runs: its 14-day lifetime is shorter than a month.

**Health warnings block publishing.**
`jobradar monthly` exits with code 1 when a source returns nothing, failures run high, or a role's figures jump. Those are the signs that a board changed its markup or started blocking the crawler. The workflow then opens an issue and publishes nothing. After review, re-run the workflow with mode `publish_only` to load the committed snapshot.

**D1 publishes atomically.**
Each snapshot loads with status `loading`, and its last statement flips it to `published`. Readers query `current_snapshot`, never "the newest row", so a failed load never shows. Older snapshots stay in the database for month-on-month views.

**The radar has its own D1 database.**
The monorepo's `packages/db` owns the schema for community data (`community-tools`). The radar writes whole snapshots on its own schedule, with its own migrations in `d1/migrations/`. Keeping it in a separate `job-radar` database means neither can break the other. Frontends only read it. This repo should publish TypeScript types for the tables as the contract.

**External pages are static.**
The data changes monthly and is the same for every visitor, so querying D1 per page view repeats the same work. Prospects arrive from search, so numbers in the HTML help SEO and load time. The site never holds database credentials at runtime, so postings and employer names can't leak. In Astro, keep `output: "server"` and set `prerender = true` on the radar pages. Read D1 at build time through the D1 HTTP API with a read-only token. Interactivity is client-side over data already in the page: switching between Local and Global, or between roles. Lead capture posts to an existing write endpoint.

**Internal pages are dynamic where they need to be.**
Staff will filter by employer, search postings and compare months: real queries, so these pages render per request against a D1 binding. Pages that only show charts can still be prebuilt. Cloudflare Access restricts the site to Zindua accounts. It is free for up to 50 users.

## What each frontend may read

| Data | External | Internal |
| --- | --- | --- |
| Role tables (`role_market_summary`, `role_market_months`, `role_market_skills`, `role_market_pay`) | Yes | Yes |
| Skill and programme tables (to add; see below) | Yes | Yes |
| `caveats`, `snapshots`, `markets` | Yes | Yes |
| Posting-level or employer-level detail | No | Yes |

Rules for every view: show shares, not raw counts. Hide figures below the minimum sample (100 postings for skill trends, 30 per role, 10 for pay). Never combine Kenyan and global figures. Never convert KES to USD.

## Gaps to close

1. **Skill tables in D1.** D1 holds only role tables today. The design-system dashboards read `skill_diffusion.csv` through `reports/skill_data.js`. Add skill and programme tables to the snapshot so both frontends use one source.
2. **Internal data in D1.** Posting-level tables for drill-downs. Keep them out of the external build's queries.
3. **pandas 3.** The role and monthly exports fail under pandas 3.x. The workflow pins `pandas<3` until that's fixed.
4. **Monorepo docs.** Its `CLAUDE.md` describes `apps/job-radar` as SSR fed by a planned `workers/jobs-ingest` Worker, and puts internal staff tools in `people-portal`. Update both points to match this note.

## Setup checklist

- [ ] Merge `job_skills_radar` into this repo (MERGE_PLAN.md).
- [ ] Create the R2 bucket `job-radar` and upload the current `data/raw/jobs.sqlite` to `raw/jobs.sqlite`.
- [ ] Create an R2 API token scoped to that bucket (Object Read & Write).
- [ ] Create the D1 database `job-radar` and a Cloudflare API token with D1 Edit.
- [ ] Add repository secrets: `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN`.
- [ ] Add repository variables: `R2_BUCKET=job-radar`, `D1_DATABASE=job-radar`.
- [ ] Create a deploy hook in each Pages project and add `EXTERNAL_DEPLOY_HOOK` and `INTERNAL_DEPLOY_HOOK`.
- [ ] If `master` is protected, let `github-actions[bot]` push, or change the workflow to open a pull request instead.
- [ ] Run the workflow by hand with mode `skip_collect` to test everything except the crawl.
- [ ] Put the internal site behind Cloudflare Access.
