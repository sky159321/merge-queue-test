# Merge Queue Smoke Test — Handoff & Plan

**For:** a fresh Claude Code session started in this repo (`sky159321/merge-queue-test`).
**From:** a monorail session where Jake (skim@bridge.xyz) + Alex designed Bridge's
merge-queue rollout. This doc is self-contained — you have no prior context, so
everything you need is here.

---

## 0. How to use this doc

You are working in a **throwaway public test repo** whose only purpose is to
**empirically verify GitHub's native merge-queue behavior** before we touch the real
monorail. Nothing here ships to production. You may create branches, open PRs, enable
the merge queue, and merge freely.

- Prefix branches with `claude/` (Jake's global git rule allows pushing/PR-ing those).
- All `gh` commands must be prefixed `GH_HOST=github.com` (Bridge's default `gh` points
  at internal GHE; this repo is on public github.com).
- Default branch is **`main`** here (monorail uses `master` — don't get confused).
- Work through the scenario matrix in §8, record observed behavior in the §9 table,
  and produce the findings summary in §11. That's the deliverable.

---

## 1. Mission

Prove out **GitHub native merge queue** on a toy repo with **deterministic,
reproducible** scenarios (not random flakes), so we know exactly how it behaves before
enabling it on monorail. Alex's key technique: **use `sleep` in CI so concurrent PRs
reliably interleave** — don't try to reproduce timing races with randomness.

---

## 2. Why — the monorail project this feeds

Bridge's monorail has a fast-moving `master`. Goals of adopting a merge queue:
- Stop broken commits landing on `master` (every queued PR is tested against
  `master` + everything ahead of it *before* merging).
- Survive flaky tests without thrashing the queue.

**Decision (made):** use **GitHub native merge queue**, **batch size 1**, **speculative
parallel** builds — chosen over a Bors-style bisect queue because it favors **wall-clock
time** over compute savings. Alex lived through "bisect hell" with flaky tests at a prior
company using a bisect queue; that's the motivation for solid flake handling *before*
flipping the switch.

The speculative model (this is what we're verifying):
```
PR1 build = main + PR1
PR2 build = main + PR1 + PR2      (stacked on PR1)
PR3 build = main + PR1 + PR2 + PR3
```
All run concurrently. Happy path = ~zero wall-clock penalty vs today. If PR1 fails, PR2/PR3
rebuild off the new head; PR1 is ejected back to its author.

---

## 3. Decisions already made (don't relitigate — just verify)

- **Merge method:** squash.
- **Batch size 1** (`max_entries_to_merge: 1`), **build concurrency ~5**
  (`max_entries_to_build`).
- **Flaky-test retry:** use the **`rspec-retry` gem (in-process retry)** + a distinct CI
  log line so recovered flakes stay observable. (An earlier fresh-process/subprocess
  prototype was rejected as over-complicated; in-process handles every flake category
  except rare same-shard global-state pollution, which is rare and easy to root-fix.)
  For this repo, model the retry with any test framework — the point is to confirm the
  *queue* treats a retried-then-green check as green.
- **Treat the merge-queue branch (`gh-readonly-queue/*`) as equivalent to `master`** for
  all CI steps **except the final deploy trigger**.
- **Move Docker build+push into the merge-queue run** (not the post-merge master run) so
  it's parallelized. (Here: model Docker as an `echo` step.)
- **Master run is skipped for queue-produced commits** but **must still run in full for
  force-merges** that bypass the queue → we need a way to *detect* a force-merge.

These monorail decisions are *why* the scenarios in §8 matter — each scenario confirms an
assumption one of these decisions rests on.

---

## 4. GitHub native merge queue — how it works + config knobs

When a PR is approved and you click **Merge when ready** (or `gh pr merge --auto`), GitHub:
1. Builds a temporary branch `gh-readonly-queue/main/pr-<n>-<sha>` = `main` + PRs ahead +
   this PR.
2. Dispatches a **`merge_group`** webhook/event. Your CI must trigger on `merge_group` or
   no checks report on the group.
3. Waits for the **required status checks** to pass **on the merge-group commit**.
4. On success, fast-forwards `main` to that commit; the PR leaves the queue.

`merge_group` is a **separate event** from `pull_request` and `push`. A workflow without an
`on: merge_group` trigger produces **no** checks on the group → the queue waits, times out,
and ejects. (That's a wedge to watch for.)

**Config knobs** (UI: Settings → Rules/Branches → *Require merge queue*; API: a
`merge_queue` rule in a ruleset):

| Knob | Meaning |
|---|---|
| `merge_method` | merge / squash / rebase. We want **squash**. |
| `max_entries_to_merge` (Max PRs to merge) | batch size. **1**. |
| `min_entries_to_merge` + `min_entries_to_merge_wait_minutes` | wait for N entries or this long before forming a group. |
| `max_entries_to_build` (Build concurrency, 1–100) | how many `merge_group` builds dispatch in parallel. Set **>1** (e.g. 5) or you can't observe stacking. |
| `grouping_strategy` | **ALLGREEN** = "Only merge non-failing PRs" **ON** (every PR must pass). **HEADGREEN** = OFF (a failed PR can ride in a group as long as the **last** PR in the group passed, i.e. the *combined* diff is green). |
| `check_response_timeout_minutes` | how long to wait for checks before treating them as failed. |

The **"Only merge non-failing pull requests"** setting in the UI == `grouping_strategy`
(`ALLGREEN` ↔ checked, `HEADGREEN` ↔ unchecked). Testing both is scenario **A2**.

Example ruleset rule (for `gh api`, if you prefer API over UI):
```json
{
  "type": "merge_queue",
  "parameters": {
    "merge_method": "SQUASH",
    "max_entries_to_merge": 1,
    "min_entries_to_merge": 1,
    "min_entries_to_merge_wait_minutes": 1,
    "max_entries_to_build": 5,
    "grouping_strategy": "ALLGREEN",
    "check_response_timeout_minutes": 15
  }
}
```

---

## 5. This repo's current state

- `sky159321/merge-queue-test`, **public**, default branch **`main`**, one commit, remote
  wired (`git@github.com:sky159321/merge-queue-test.git`).
- No CI workflow yet. No merge queue enabled yet.
- Merge queue is available on public repos, so no plan upgrade needed.

---

## 6. Setup steps

### 6a. Add a minimal CI workflow

Create `.github/workflows/ci.yml`:

```yaml
name: ci
on:
  pull_request:
  merge_group:
  push:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run scenario verdicts
        run: |
          set -uo pipefail
          echo "event=${{ github.event_name }} ref=${{ github.ref }} sha=${{ github.sha }}"
          overall=0
          shopt -s nullglob
          for f in scenarios/*.sh; do
            SLEEP=0; EXIT=0
            # shellcheck disable=SC1090
            source "$f"
            echo "::group::$f (sleep=$SLEEP exit=$EXIT)"; cat "$f"; echo "::endgroup::"
            sleep "$SLEEP"
            [ "$EXIT" -ne 0 ] && overall=1
          done
          exit $overall
      - name: Docker build+push (echo only, merge_group only)
        if: ${{ github.event_name == 'merge_group' }}
        run: echo "pretend docker build+push for ${{ github.sha }}"

  all-green:
    needs: [check]
    if: ${{ always() }}
    runs-on: ubuntu-latest
    steps:
      - run: |
          [ "${{ needs.check.result }}" = "success" ] && echo "all green" || exit 1
```

Make **`all-green`** the single required status check (an aggregator, mirroring monorail's
"All Checks Passed."). Commit this to `main` first.

### 6b. Enable branch protection + merge queue on `main`

Easiest via UI: Settings → Branches → Add branch protection rule for `main` →
- Require status checks: add **`all-green`**.
- **Require merge queue** (set merge method = squash, build concurrency = 5, and toggle
  "Only merge non-failing pull requests" per the scenario you're testing).

Or via `gh api` rulesets (see §4 JSON + a `required_status_checks` rule for `all-green`).

### 6c. The deterministic test harness

Each test PR adds a file `scenarios/<name>.sh` containing its verdict:
```bash
# scenarios/pr1.sh
SLEEP=30   # keeps this build running so later PRs enter the queue concurrently
EXIT=1     # 0 = pass, 1 = fail
```
Because a merge group is `main + PRs ahead + this PR`, the merge-group checkout contains
**all** the `scenarios/*.sh` from the stacked PRs, so CI evaluates the *combined* set —
exactly the semantics we want to probe. Big `SLEEP` on early PRs is how you force reliable
interleaving.

For a **semantic-conflict** test, have two PRs edit the **same line** of a shared file
(e.g. both change `conflict.txt`) so each passes alone but the second can't cleanly build on
the first.

---

## 7. How to drive PRs into the queue

```bash
GH_HOST=github.com gh pr create --base main --head claude/pr1 --title "pr1" --body "..."
GH_HOST=github.com gh pr merge <n> --auto --squash   # adds it to the queue
```
Open several quickly so they stack. Watch the queue in the PR UI / Actions tab, and observe
the `gh-readonly-queue/main/*` refs and their `merge_group` runs.

---

## 8. Scenario matrix (run these, record what you see)

| # | Scenario | Setup | What to observe / confirm |
|---|---|---|---|
| **A1** | Speculative stacking + failure isolation | PR1 (`EXIT=1, SLEEP=30`), PR2 (`EXIT=0, SLEEP=30`), PR3 (`EXIT=0`), all independent files; enqueue all 3 fast | 3 concurrent `merge_group` builds stacked; PR1 ejected; PR2/PR3 **rebuild off new head** and merge |
| **A2** | "Only merge non-failing" toggle | Same as A1 but run once with the toggle **ON** (ALLGREEN) and once **OFF** (HEADGREEN) | Does a group with a failing earlier PR still merge if the **last** PR passes? Document the exact difference |
| **A3** | PR1-fail / PR2-pass notification | From A1 | Is PR2 ever merged **alone**, or always rebuilt off new head? **How + when is PR1's author notified** — permanent eject-and-resubmit, or "temporarily removed, will merge if a later group passes"? |
| **A4** | Merge method | squash configured | Confirm squash; confirm the merged commit == the tree tested on the merge group |
| **A5** | Build concurrency | set `max_entries_to_build=5`, enqueue 5+ PRs | Confirm up to 5 `merge_group` builds dispatch at once |
| **A6** | Docker-on-queue-branch | the `echo` docker step gated `if: merge_group` | Confirm it runs on the merge-group build and **not** on the `push`-to-main run |
| **A7** | Force-merge detection | push a commit **directly to `main`** (bypass queue), or `gh pr merge --admin` | **What signal distinguishes a force-merge from a queue merge on the `push` event?** (commit ancestry via `gh-readonly-queue/*`? pusher? a marker?) This is the unsolved one — find a reliable detector |
| **A8** | In-process retry stays green | a scenario that fails first attempt then passes (e.g. a marker file the CI step creates), using an in-process retry | Confirm the queue sees the retried-green check as green and merges; confirm a distinct "flake recovered" log is emittable |

Also worth noting while you're in there:
- Does the **`push`-to-`main`** run fire after a queue merge (it should — the queue
  fast-forwards `main`)? This is monorail's deploy-trigger question.
- What does a **stale/timed-out** required check do to the queue (wedge vs eject)?

---

## 9. Deliverable: observed-behavior table (fill this in)

| Question | Setting | Observed behavior | Notes |
|---|---|---|---|
| PR1 fails, PR2 stacked passes → does PR2 merge alone or rebuild? | | | |
| How/when is PR1's author notified on eject? | | | |
| "Only merge non-failing" ON (ALLGREEN) behavior | | | |
| "Only merge non-failing" OFF (HEADGREEN) behavior | | | |
| Does `push`-to-main CI run after a queue merge? | | | |
| Force-merge vs queue-merge — reliable detector | | | |
| Merged commit == tested merge-group tree? | squash | | |
| Retried-green check treated as green by queue? | | | |
| Timed-out required check → wedge or eject? | | | |

---

## 10. Conventions & constraints

- `GH_HOST=github.com gh ...` for every `gh` call.
- Branches: `claude/...`; pushing them + opening PRs is fine here.
- Public repo → no secrets, no Bridge proprietary code. Toy scripts only.
- Prefer the UI for enabling the merge queue if the API is fiddly; both work.
- This is a spike — bias toward getting observations fast over polish.

---

## 11. Definition of done

1. §9 table filled with observed behavior for every row.
2. A short **findings** section (append here) translating each observation into a monorail
   implication — especially: the force-merge detector (A7), the "only merge non-failing"
   choice (A2), and whether the post-merge `push` run fires (deploy trigger).
3. Any surprises or GitHub quirks worth warning the team about.

This findings output feeds Jake's engineering design doc for the monorail rollout.

---

## Appendix — monorail context you may be asked about

The monorail-side rollout (separate work, not in this repo) is sequenced:
1. rspec-retry (in-process) + recovered-flake logging.
2. `ci.yml` `merge_group` wiring + audit of **66 `master` references** (classify each:
   runs on merge-queue branch / stays master-only / force-merge path).
3. Move Docker push to the merge-queue run; **re-point the deploy trigger** (deploy today
   fires on `ci`-completed-on-`master`; if the master test run is skipped for queue commits,
   deploy must be re-pointed or it silently stops).
4. Verify Socket (a 3rd-party required check) reports on `gh-readonly-queue/*` commits.
5. Flip the merge-queue ruleset rule last (instantly revertable).

Your smoke-test findings de-risk steps 2–4.
