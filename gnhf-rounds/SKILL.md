---
name: gnhf-rounds
description: "Run parallel autonomous coding agents with gnhf in a zellij tab or tmux window, one pane and one git worktree per agent, then review, merge, and repeat in rounds until the work is done. Use when the user asks to run gnhf, spin up/fan out agents in zellij or tmux panes, 'run agents overnight', 'do this with gnhf instances', or to keep a review→merge→next-round loop going."
---

# gnhf rounds in zellij / tmux

`gnhf` runs a coding agent in iterations, committing after each one ("good night, have fun"). This skill runs several
gnhf agents at once, one pane each, each in its own git worktree, and drives them in **rounds**:

**brief → launch → watch → merge → review → next round**, until reviews come back clean.

The launcher is `scripts/launch-round.sh` (next to this file). Copy it into the repo as `ops/agents/launch-round.sh`
(or call it by its full path), because panes re-run it.

## 1. Write the briefs (committed, or worktrees can't see them)

```
ops/agents/<round>/00-common.md     shared rules for every agent in the round
ops/agents/<round>/1-<slug>.md      one brief per agent (n = 1..9)
ops/agents/<round>/reviews/*.md     review findings the briefs point at (round 2 onwards)
```

`00-common.md` must say:
- **What the system is.** Paths, how to run tests, which branch it's on.
- **Hard rules:**
  - nothing sent externally;
  - never print, log or commit secrets, and run gitleaks before each commit;
  - no CAPTCHA bypass;
  - no global installs (`brew`, `npm -g`, `sudo`);
  - don't touch unrelated containers.
- **Docker safety in worktrees.** Every checkout of a repo resolves to the same compose project name as the live
  stack. So a worktree must always use `docker compose -p <unique-test-name>` on other ports, never the live
  project, and tear its test stack down. Only one designated agent, or the coordinator, touches the live stack.
- **Ownership.** Which files each agent owns. Give shared files (a big compose file, a bootstrap script) marked
  sections per agent, or one file per agent, so merges don't conflict.
- **The working loop:**
  - keep a status file `ops/agents/status/<round>-<n>.md` as a checklist with evidence;
  - commit small, working steps;
  - finish with the exact last line `ALL DONE`, which is the stop condition.

Each agent brief says: the goal, the files it owns, the steps, how to verify it (tests and real checks), and which report
file to write (`ops/agents/REPORT-<round>-<n>.md`).

## 2. Launch

From the repo root, on a base or integration branch (never a `gnhf/*` branch; gnhf refuses `--worktree` there):

```sh
git add ops/agents/<round> && git commit -m "Round <round> briefs"
GNHF_STAGGER=30 path/to/launch-round.sh <round> <tab-name>    # opens a zellij tab or tmux window, one pane per brief
path/to/launch-round.sh <round> run 2                          # one agent in the current terminal
```

The launcher:
- refuses to start from a gnhf branch, with uncommitted briefs, or when the round is already running;
- staggers starts, so worktree creation doesn't race on git locks;
- caps each agent: `GNHF_MAX_TOKENS`, `GNHF_MAX_ITER`, `--max-rate-limit-wait 3h`;
- uses zellij by default, or tmux when inside tmux; `GNHF_MUX` forces either.

The worktrees land in `../<repo>-gnhf-worktrees/<slug>/` on branches `gnhf/<slug>`. `git worktree list` maps them.

## 3. Watch

- **One command:** `launch-round.sh <round> status` prints, per agent, whether it is alive, its last commit age and its status file's last line.
- **By hand:** `pgrep -f "You are round <round> agent"`, `git log -1 --format='%h %cr' <branch>`, `tail -1 <worktree>/ops/agents/status/<round>-<n>.md`.
- **Phone pings:** each agent's notifications are named `gnhf <round> agent <n>` and carry repo and branch (the `agent-notify` skill).
- **Stuck:** no commit for about 60 minutes while the process is alive. When everyone stalls at once, suspect a usage limit.
- **For long runs, schedule check-ins** (a cron or loop tool) every 30–120 minutes. Each check-in reads status, then
  merges, reviews or launches.

## 4. Merge, review, next round

1. **Merge finished branches into an integration branch** (e.g. `integrate/rN`) in its own worktree. Leave the
   branch a live agent is committing to alone. Then run the full test suite and gitleaks.
2. **Review with 2–3 read-only reviewer subagents in parallel**, one per area. Ask each for FIXED / NOT FIXED per
   earlier finding, new findings ranked BLOCKER / SHOULD-FIX / NIT with file:line, failure scenario and fix, and a
   clear verdict. Save each review to `ops/agents/<next round>/reviews/`.
3. **If anything is BLOCKER or SHOULD-FIX:** write the next round's briefs from the reviews, commit them on the
   integration branch, and launch from that checkout.
4. **When reviews are clean:** fast-forward the base branch, then do the end-to-end check yourself (browser,
   real logins, real data counts).

## 5. Clean up when a round is merged

`launch-round.sh <round> prune <integration-branch>` removes every finished worktree and `gnhf/*` branch whose commits are
all inside `<integration-branch>` and that has no uncommitted files. It refuses while the round's agents run, keeps anything
unmerged, and saves each deleted branch name and commit to `../<repo>-gnhf-worktrees/pruned-branches-<date>.txt`.

## Cost: agents are expensive, so size the round

Every iteration re-reads the whole brief and repo context, so a 30-minute agent can burn 7–15M tokens (mostly cache reads),
and all agents share one account's limits. Before launching, ask whether a plain script does the job:
- **Data and content generation** (seed data, fixtures, copy): write a deterministic script. No agents needed.
- **Independent code changes**: 3–5 agents, one per area, with `GNHF_MAX_ITER` and `GNHF_MAX_TOKENS` lowered.
- **A review round** after a merge: 2–3 read-only reviewer subagents, not more gnhf agents.
Stop when reviews are clean or the user says it's costing too much; the work is committed, so stopping loses nothing.

## Gotchas learned the hard way

- **A signed-out agent dies silently** (no commits, no error). The launcher checks `claude auth status` first.
- **gnhf resets uncommitted files when an iteration fails** (e.g. after a usage-limit stall). Anything you hand an
  agent mid-run must be **committed**, not just written to its status file.
- **gnhf processes stay alive, idle, after `ALL DONE`.** Close their panes; don't treat them as working.
- **A glob like `[0-9]*-*.md` also matches `00-common.md`.** Agent briefs start at `1-`.
- **Every agent in every pane shares the same account limits.** A usage limit pauses all of them at once.
- **A worktree has no git-ignored files** (`.env`, secrets). An agent that needs them must run in the main checkout
  (`--current-branch`), and only one agent may.
- **`docker compose up -d` doesn't recreate containers whose config didn't change.** After switching a stack's
  layout, `--force-recreate` its background workers.
