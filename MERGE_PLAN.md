# Merge plan: `job_skills_radar` into `job-market-skills-radar_new`

**Status:** proposal, for review by Chris before anything is pushed.
**Date:** 3 November 2026

## Why merge

We have two GitHub repos for the same project, both named `jobradar`:

| | `job-market-skills-radar_new` (this repo) | `job_skills_radar` |
|---|---|---|
| History | 27 commits, from Phase 1 (6 Sep) to the ponytail audit (3 Oct) | 4 commits, all 18 Sep, starting with one "Initial commit" |
| Strengths | Two dashboards on the Zindua School Design System; trimmed dependencies | Role classification and pay; Career Point Kenya history; D1, Sheets and Looker publishing; `monthly` run; ~50 more skills; ~1,050 company boards |

Neither repo contains the other. Fixes to the shared core (fetching, JSON-LD parsing, skill extraction) now have to be made twice, and the two drift further apart every week.

## What we found

`job_skills_radar` is a branch of this repo that lost its history. Its "Initial commit" (`020c8cd`) is this repo's `e425236` *"Phase 7: curriculum briefs and the published report"* plus about 7,700 added lines, and only about 100 lines differ the other way. No other commit here comes as close.

```
Phase 1 … Phase 7 (e425236, 6 Sep)
   ├── job_skills_radar: roles, pay, Career Point Kenya, D1, monthly, site   (4 commits, link lost)
   └── this repo: design-system dashboards, PRs #1–3, ponytail audit        (20 commits)
```

Because of that shared starting point, the two can be merged as normal branches rather than as unrelated repos.

## The plan

1. **Restore the lost link.** In a clone of this repo, fetch `job_skills_radar` and make `e425236` the parent of its first commit:
   ```bash
   git fetch https://github.com/zinduaofficial/job_skills_radar.git HEAD:old
   git replace --graft $(git rev-list --max-parents=0 old) e425236
   git filter-repo --refs old --force   # or filter-branch; makes the graft permanent
   ```
   This gives those 4 commits new IDs in this repo only. The originals stay untouched in `job_skills_radar`.
2. **Merge on a branch.** `git switch -c merge/job-skills-radar && git merge old`, then resolve conflicts (see below).
3. **Check it.** Run the tests and lint, regenerate the outputs and dashboard data, and confirm the dashboards render.
4. **Review.** Open a pull request into `master` here.
5. **Archive the other repo.** After merging, add a README note to `job_skills_radar` pointing here and archive it on GitHub. No force-push anywhere.

## Dry-run results

We ran steps 1–2 in a scratch copy (nothing pushed):

- **Common ancestor found:** `e425236`, as expected.
- **Only 2 files conflict:**
  - `PLAN.md`: this repo keeps the row *"Is Cybersecurity Core sized right?"* and the note *"Confirmed as matching your current programme list"*; `job_skills_radar` drops them. One-line call.
  - `pyproject.toml`: `job_skills_radar` adds the `sheets` (gspread) and `notebooks` optional extras. Keep both; `jobradar monthly --sheets` needs gspread.
- **Merged automatically:** 45 files added from `job_skills_radar`, 22 changed, 2 deleted.
- **Tests:** 518 pass and 20 fail (`test_role_outputs.py`, `test_monthly_exports.py`). The same 20 fail in `job_skills_radar` on its own, so the merge didn't cause them. They break under pandas 3.x (we had 3.1.0), which raises `TypeError: Can only string multiply by an integer` on the new string dtype. Fix the code, or pin `pandas>=2.2,<3` until it's fixed.
- **Lint:** `ruff check` passes.

## Decisions for Chris

1. **Dropped dependencies.** The ponytail audit removed `pyarrow` and `tenacity`. The merged code no longer imports either, so it's fine to keep them dropped. But `job_skills_radar` still commits `data/processed/*.parquet` and `cooccurrence_*` files that nothing produces any more. Delete them?
2. **Curriculum outputs.** `job_skills_radar` deleted `reports/curriculum_briefs.md` and `reports/track_coverage.csv` as stale, but `pipeline.py` still writes them and `skills_radar_dashboard.html` quotes the briefs. Regenerate them, or have the dashboard read the data directly?
3. **Which dashboards are live?** `job_skills_radar` has `reports/dashboard.html` (internal) and `reports/skills-analysis.html` (public lead magnet). This repo has `skills_radar_dashboard.html` and `skill_trends_dashboard.html`. Keep all four, or move the first two onto the design system too?
4. **Anything deployed?** Does any live system point at `job_skills_radar`, such as the D1 database, the Google Sheet, Looker Studio or a hosted page? If so, move it to this repo before archiving.
5. **Was the split deliberate?** For example, a public repo and a private one. If so, stop here and keep two repos with a shared core instead.

## After merging

- Rebuild `reports/skill_data.js` with `scripts/build_dashboard_data.py` from the merged data, so the dashboards include the role and Career Point figures.
- Combine the two READMEs: this repo's dashboard section plus `job_skills_radar`'s data-cleaning and publishing sections.
- Delete this file.
