---
name: pr-description
description: The required structure for well-written pull request descriptions in any repository. Covers single-branch repos where every PR targets main and repos with integration and release branches, and requires the manual server-side steps the author must apply after merge (database migrations, Supabase RLS/Vault/edge functions, Firebase, env vars, client releases). Use whenever you write, open, or update a PR description, PR body, or PR summary, including when a PR is created by a pipeline or with `gh pr create` / `gh-axi pr create`.
user-invocable: true
---

# Structured PR descriptions

A good PR description lets a reviewer get the point from the Summary table alone, then read on only for what they need.
It also tells the author, in exact terms, what must still be applied by hand after the merge - because merging code changes nothing until the database, secrets, functions, and config are updated too.

## First, find the repo's branch model

Check the repo's `CONTRIBUTING.md`, `AGENTS.md`, and recent merged PRs, then pick one of two shapes:

- **Single-branch** (e.g. every PR targets `main`): there is no integration or release branch, so every PR uses the feature/fix structure below, and the human steps go in the required **Manual steps** section.
- **Staged**: feature and fix PRs merge into an integration branch (commonly `develop`), and a separate release or pre-release PR bundles them into a stable branch (`staging`, `main`, `production`) from a `release/*` or `pre-release-*` branch. Use the feature/fix structure below for the former and the release structure further down for the latter.

If the repo is single-branch, do not invent a `develop` or `staging` shape for it: the feature/fix structure plus **Manual steps** is the whole PR.

## Feature and fix PRs

### Sections, in this order

1. `## Summary`: a two-column table with an empty header (`| | |`), one bold label per row, one or two plain sentences per cell.
   - For a fix: **Problem** (what the user saw), **Cause**, **Fix**, **Proof**, **Risk**, **Needs the team**.
   - For a feature: **Problem**, **Fix**, then rows such as **Where** (where it shows up) or **Migration**, then **Proof** and **Needs the team**.
   - Add a row like **Cost** only when it matters.
   - **Proof** gives real numbers from a real run: before on the base branch and after on this branch.
   - **Needs the team** says what reviewers or other repos must do, and points at the Manual steps section for anything applied by hand.
     Other-repo dependencies go here as repo, number and title, for example `acme-backend #449: feat(api): record per-turn phase timing`.
     A needed PR that does not exist yet is written as missing, for example `acme-backend: missing - speech to text fallback chain`.
     With nothing needed, say so: "Nothing before merging."
2. `## Manual steps` (required): every action a human must take after this PR merges for the change to be live, in order. **Prefer one reproducible, idempotent script that runs the whole sequence end to end**, with any secret or password supplied at run time, over a list of loose commands run one at a time; when a step truly cannot be scripted, give its exact command or setting. This is the server and deployment side, for example:
   - Database: run migrations (e.g. `supabase db push`), backfills, or any RLS/policy change.
   - Supabase: store a secret in Vault, set an Edge Function secret, deploy functions, change auth settings.
   - Firebase or other third parties: add a config value, upload a key, publish a rule.
   - Environment: add or change a variable, in a local file and in the deploy environment.
   - Client: release an app build that matches a schema change.
   Name any ordering constraint that matters, e.g. "set the secret before the functions take traffic".
   When nothing must be applied by hand, keep the heading and write `- None - nothing to apply by hand.` Do not drop the section: its absence hides the question.
3. `## Decisions`: a table with columns `Decision | Options considered | Choice | Why | Pros | Cons`, one row per real choice.
   Credit a choice the requester or team made (for example "requester, D7").
4. `## Intent`: the request in the requester's own words (quote them, with the date), plus the later decisions that shaped this PR.
5. `## What Changed`: bullets, one per change, naming the files or components touched and the behavior that follows.
6. `## Risk Assessment`: one short paragraph opening with a level, e.g. "✅ Low:", saying what could break and why it will not.
7. `## Testing`: how it was driven end to end (real stack, real providers, browser, data used), what passed, what was not tested and why, and cleanup done.
   Add a results table or evidence in `<details>` blocks when there is more than a few lines.

If a delivery pipeline appends its own section to the PR body, leave it alone and never write it by hand.

### Template

```markdown
## Summary

| | |
|---|---|
| **Problem** |  |
| **Cause** |  |
| **Fix** |  |
| **Proof** |  |
| **Risk** |  |
| **Needs the team** |  |

## Manual steps

- [ ] Step a human must do after merge, with the exact command. (#NNN)

## Decisions

| Decision | Options considered | Choice | Why | Pros | Cons |
|---|---|---|---|---|---|
|  |  |  |  |  |  |

## Intent

## What Changed

-

## Risk Assessment

## Testing
```

### Example (no manual steps)

```markdown
## Summary

| | |
|---|---|
| **Problem** | Meetings booked in zones with a half-hour offset (e.g. Asia/Kolkata) showed the wrong start time in the confirmation email. |
| **Cause** | The email renderer formatted the stored UTC timestamp by adding a whole-hour offset, dropping the 30-minute component. |
| **Fix** | Format the start time with the booking's stored IANA timezone instead of a fixed hour offset, and cover half-hour and 45-minute zones in tests. |
| **Proof** | Real stack with a local mail catcher: 5 of 5 half-hour-zone bookings emailed the wrong time on the base branch; all 5 correct to the minute on this branch. Whole-hour zones unchanged. |
| **Risk** | Low: a formatting-only change; the stored instant is untouched and existing timezone tests still pass. |
| **Needs the team** | Nothing before merging; nothing to apply by hand. |

## Manual steps

- None - nothing to apply by hand.

## Decisions

| Decision | Options considered | Choice | Why | Pros | Cons |
|---|---|---|---|---|---|
| Where to fix the offset | patch the hour arithmetic; format with the stored IANA zone | format the IANA zone | The zone is already captured at booking, so the arithmetic is the wrong layer | Correct for every zone, including DST | Slightly larger change |

## Intent

Requester, 12 Mar 2026: "the confirmation email shows the wrong time for the India team".
Follow-up decision D2: fix it at the formatting layer rather than patching the arithmetic, so every zone is handled.

## What Changed

- `emails/render.py`: format the booking start with its stored IANA timezone, replacing the fixed whole-hour offset.
- `tests/test_email_times.py`: new cases for +05:30 and +05:45 zones and one DST transition.

## Risk Assessment

✅ Low: only the display path changed; the stored instant, the API response, and existing whole-hour timezone tests behave as before.

## Testing

Drove the real booking flow on a local stack with a real mail catcher, booking the same slot from UTC, Asia/Kolkata (+05:30), and Asia/Kathmandu (+05:45).
On the base branch the half-hour and 45-minute bookings emailed times off by 30 and 45 minutes; on this branch all match the booking page to the minute.
Whole-hour zones and the DST-boundary booking are identical on both branches. No UI change, so the evidence is email bodies, not screenshots.
```

### Example (with manual steps)

```markdown
## Summary

| | |
|---|---|
| **Problem** | The "recent meals" list takes several seconds on large tables because it sorts without a usable index. |
| **Fix** | Adds a composite index on `(table_id, created_at desc)` and serves the list through it. |
| **Proof** | Real database with 200k meal rows: the page query drops from ~2.1 s to ~40 ms; EXPLAIN now shows an index scan instead of a sort. |
| **Risk** | Low: additive index; the query result is unchanged, only its plan. |
| **Needs the team** | Apply the migration after merge (see Manual steps). |

## Manual steps

- [ ] Apply the migration: `supabase db push` (adds index `meals_table_created_idx`).
- [ ] Confirm it applied: `select indexname from pg_indexes where tablename='meals';`
- [ ] No app release needed; no downtime expected.

## Intent

Requester, 3 Apr 2026: "the meals list feels slow once a table has a lot of meals".
```

## Release and pre-release PRs

Only for a staged/branch model. In a single-branch repo there is no release PR; skip this section.
Title: `Pre release vX.Y.Z` (or `Release vX.Y.Z`).
Top-level `#` headings, in this order:

1. `# Changes`: one bullet per included PR, opening with a bold one-line outcome in plain words, then one or two sentences of what changed, ending with the PR number, e.g. `(#471)`.
   Housekeeping goes in a last plain bullet.
2. `# Manual steps` (required): a checklist of every action a human must take on the target environment, each with its PR, following the same rules as the feature/fix Manual steps section - migrations, Vault and Edge Function secrets, function deploys, Firebase or third-party config, env vars, client releases, with ordering constraints called out, and preferring one reproducible script over loose commands.
   With none, write `- None`.
   Find them in the diff against the target branch: new migrations, new or changed keys in `.env.example` or other config samples, code reading new env vars, and workflow/config changes.
3. `# Catch-up merge`: how the branch was cut (from the integration branch) and caught up with the target (by merge, not rebase), every conflict with the side kept and why, and any fix the combined tree needed, with its commit.
4. `# Tests` (or `# Build and tests`): what ran on the merged tree and the result, e.g. "254 tests, OK" with the CI run; name external flakes as such.
5. `# New Migrations`: when there are any; each migration with its PR and whether it rewrites data.
6. `# Checks on staging` (or the environment's name): first the paired release PR in any other repo, as a full URL, and the merge order it needs (write "missing" if it does not exist yet); then short manual checks to do after deploy, including real-device checks for UI changes.

### Release template

```markdown
# Changes

- **Outcome in plain words.** What changed. (#NNN)

# Manual steps

- [ ] Step to do on the environment (migration, secret, function deploy, config). (#NNN)

# Catch-up merge

# Tests

# Checks on staging

- Paired release PR: https://github.com/<org>/<repo>/pull/NNN, merge order.
-
```

### Manual test page

For a change that needs manual steps, build a Lavish page with the steps and tests, and open it with the `lavish` skill (`lavish-axi <file>`).
Start from `release-tests-template.html` next to this file. Its shape:

- a Setup / manual-steps section first, listing every item from the PR's `Manual steps` in order as checkboxes, with the exact command in the hint - this is what the author ticks off while applying nothing yet;
- then one section per area, naming the PRs it covers;
- one checkbox per concrete check, with a short how-to hint where it is not obvious;
- a progress bar, and checkbox state saved in the browser under a key unique to the page;
- a `new` tag on steps added or changed since the last round.

Make one page for the local round and one for the deployed round.
The same checks stay listed under `# Checks on staging` in a release PR.

## Screenshots (both kinds)

When the change is visible (UI, admin pages), add 1 to 3 representative screenshots, each with a short caption.
For a change with nothing on screen, prefer one readable table or chart from real numbers over raw log dumps.
Upload them straight into the PR:

- If the delivery pipeline runs a validation step that can attach media, save screenshots to its evidence directory during the test step and let it embed them in the PR it opens.
- Otherwise use plain `gh` (`gh-axi` has no `--attach`): `gh pr create ... --body-file body.md --attach ./shot.png`, with `![caption](./shot.png)` in the body where the image belongs, or `gh pr edit <url> --attach 'shot.png#caption'` for an existing PR.
- Never commit images to the repo.
- The upload needs write access to the repo; with read or triage it fails with a 404, so tell the person instead of committing the images.
- An uploaded image cannot be deleted, so check it shows nothing sensitive first.
