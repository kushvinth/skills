---
name: agent-notify
description: Ping the user's phone via self-hosted Gotify when a task or long-running job finishes, or when input is needed. Use when the user says "notify me", "ping me when it's done", "let me know when X finishes", "push me", "text me", "tell me when the build/run/deploy finishes", or when an unattended or long run (build, test suite, migration, deploy, gnhf or other agent loop, background job) completes so the user can step away. Do NOT use for in-chat status updates (just answer in chat) or for time-based reminders/scheduling (use the user's calendar or reminder tooling).
---

# Agent Notify

Send a push notification to the user's self-hosted Gotify from any agent CLI.

## Send

    ~/.agents/bin/notify.sh "<title>" "<message>"

- `title` — short event, e.g. `build finished`, `tests failed`, `deploy done`. The hostname is prefixed automatically,
  and ` · <repo>@<branch>` is appended, so the lock screen already says what it is about.
- `message` — the result in one line (e.g. `869 tests pass, 2 follow-ups left`). Optional.
- A context block is added under the message by the script: `instance` (agent, terminal pane, session), `repo` (and the
  worktree when it is one), `branch` (with uncommitted/unpushed counts) and `dir`. Do not repeat these in the message.
- Fire-and-forget: sends in the background, never prompts, exits 0 even when delivery fails. The same ping within 30 s
  is dropped. Priority is never set: Gotify uses the app's default.

Example of what arrives:

    macbook-pro: claude: finished · classicClaude@integrate/r5
    All 869 tests pass; the CRM sync fix is on integrate/r5.
    instance: claude · zellij lockin pane 22 · session 1a2b3c4d
    repo: classicClaude (worktree integrate-r5)
    branch: integrate/r5 (2 uncommitted)
    dir: ~/LocalStorage/Developer/work/startup/classicClaude-gnhf-worktrees/integrate-r5

Knobs (environment): `NOTIFY_INSTANCE="gnhf r6#2"` names the instance yourself (the gnhf launcher sets it per agent);
`NOTIFY_PLAIN=1` sends the message exactly as given; `NOTIFY_DRY_RUN=1` prints it instead of sending, to preview.

For an agent loop (e.g. gnhf), append it to the command:

    <your command>; ~/.agents/bin/notify.sh gnhf "run ended"

## When to notify

- The user asks ("notify me when …", "ping me when it's done").
- A long or unattended run finishes: build, test suite, migration, deploy, agent/gnhf loop, background job.
- A background task the user launched completes or fails.

Send one notification per meaningful event — do not notify on every step or intermediate tool call.

## Automatic stop notifications (already wired)

Claude Code (Stop/Notification hooks, with the agent's last line or permission question), Codex (`notify`, with its
last message) and opencode (`dot-config/opencode/plugins/notify.js`, on session idle) push by themselves, and gnhf via the command snippet above — no agent action needed. This skill is for **extra, on-demand** pings. If a run belongs to one of those and the user also asked to be pinged, one ping is enough.

## If nothing arrives

Credentials are cached at `~/.cache/agents/gotify.env`, hydrated once from the macOS Keychain. To re-resolve or refresh:

    ~/.agents/bin/notify.sh hydrate                                  # read Keychain (Touch ID once)
    rm -f ~/.cache/agents/gotify.env && ~/.agents/bin/notify.sh hydrate   # force refresh

Delivery attempts and skips are logged to `~/.cache/agents/notify.log`.

## Out of scope

- In-chat status updates: just reply in chat, do not push.
- Reminders, scheduling, or recurring pings: use the user's calendar or reminder tooling.
- Editing the hook wiring: hooks live in `dot-config/claude/settings.json`, `~/.codex/config.toml` and the opencode plugin; change those there, not from this skill.
