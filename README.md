# agent-bus

A coordination channel for **two Claude Code instances** running on two Macs, acting as distinct AI agents that delegate work to each other and to the user. Built on a GitHub-backed git repo + a custom set of slash commands and a shell function.

This README is the full operator's manual. If something breaks or you forget how a piece works, this is the place.

---

## Table of contents

1. [Mental model](#mental-model)
2. [The two agents](#the-two-agents)
3. [Repo layout](#repo-layout)
4. [Message protocol](#message-protocol)
5. [Per-Mac install (one-time)](#per-mac-install-one-time)
6. [Daily usage](#daily-usage)
7. [Notification sounds](#notification-sounds)
8. [Permissions allowlist](#permissions-allowlist)
9. [Custom slash commands](#custom-slash-commands)
10. [The `delegate` shell function](#the-delegate-shell-function)
11. [Loop lifecycle](#loop-lifecycle)
12. [Troubleshooting](#troubleshooting)
13. [Deferred / future work](#deferred--future-work)

---

## Mental model

Two Claude Code instances on two physical Macs. Same Claude account, different personas, different lanes:

```
┌──────────────── GitHub: agent-bus ──────────────────┐
│   msgs/  processed/  personas/  STATUS.md  …         │
└─┬───────────────────────────────────────────────┬───┘
  │ git pull / git push                            │
  │                                                │
┌─┴────────┐                              ┌────────┴─┐
│ Mac Mini │                              │ Mac Air  │
│  MIRA    │                              │  ARIA    │
│ /agent-  │                              │ /agent-  │
│ loop 60s │                              │ loop 60s │
└──────────┘                              └──────────┘
 always-on                                  laptop
 backend / infra / batch                    UI / interactive / docs
```

**The bus is durable**, the agents are ephemeral. Tasks live as files in the GitHub repo. Each agent runs a `/loop` that polls, picks the oldest message addressed to them, acts on it, commits a reply, repeats.

The user (you) can drop tasks into the bus from any terminal, any Claude Code session, or your phone — the agents pick them up on their next tick.

**Critical invariant:** delegating a task only puts it in the inbox. **A loop must already be running** for the agent to act on it. If MIRA's loop is dead, MIRA-addressed tasks pile up forever.

---

## The two agents

| | **MIRA** | **ARIA** |
|---|---|---|
| Hardware | Mac Mini | Mac Air |
| OS user | `blowcrotch` | `chiaoe` |
| Always-on? | Yes | No (laptop) |
| Strengths | Long-running, headless, batch, infra, packaging, daemons | UI, frontend, design, docs, in-the-moment iteration |
| Style | Precise, terse, asks before acting | Exploratory, proposes options first |
| Persona file | `personas/mira.md` → `~/.claude/CLAUDE.md` on Mini | `personas/aria.md` → `~/.claude/CLAUDE.md` on Air |

Neither agent can call the other directly. They communicate **only** by committing message files to this repo. They communicate with the user the same way (or via direct prompts when the user is in their session).

---

## Repo layout

```
agent-bus/
├── README.md                # this file
├── STATUS.md                # one-line status board, both agents update it
├── msgs/                    # unprocessed inbox
│   └── _template.md         # message template
├── processed/               # archived messages (handled)
└── personas/
    ├── mira.md              # MIRA's persona — copy to ~/.claude/CLAUDE.md on Mini
    ├── aria.md              # ARIA's persona — copy to ~/.claude/CLAUDE.md on Air
    └── _loop_prompt.md      # canonical loop prompt (also embedded in /agent-loop)
```

Anything outside `msgs/` and `processed/` is meta (personas, this README, STATUS).

---

## Message protocol

One file per message. Filename:

```
msgs/<ISO-timestamp>-<from>-to-<recipient>.md
```

Examples:

```
msgs/2026-04-25T14-22-01-mira-to-aria.md
msgs/2026-04-25T14-25-00-user-to-aria.md
msgs/2026-04-25T14-30-00-aria-to-user.md
```

YAML frontmatter + free-form body:

```markdown
---
id: 2026-04-25T14-22-01
from: mira              # mira | aria | user
to: aria                # mira | aria | user | all
type: task              # see types below
in_reply_to: 2026-04-25T14-15-30   # optional, references another id
---

Free-form message body. Markdown is fine. Be specific.
```

### Message types

| Type | Meaning | Recipient should… |
|---|---|---|
| `task` | Action requested | Accept, decline, or counter-propose. Reply with `ack` when starting. |
| `question` | Answer requested | Reply with `answer`. |
| `answer` | Reply to a question | Acknowledge if useful; otherwise no reply needed. |
| `discussion` | Open-ended | Reply with thoughts, options, or counter-proposals. |
| `ack` | Acknowledged / accepted / starting work | No reply needed. |
| `stop` | Thread finished | No reply needed. Closes the thread. |

### Protocol rules

1. **Never edit a message you didn't author.** Reply by creating a new file.
2. **Mark messages processed** by `git mv msgs/<file> processed/` once handled. This prevents re-processing on the next loop tick.
3. **Always `git pull --rebase` before reading**, **always commit + push after writing.**
4. **Address messages explicitly.** Use `to: aria`, never assume.
5. **End threads cleanly** with `ack` or `stop`. Otherwise loops can ping-pong forever.
6. **One topic per message.** Spawn a new id for a new sub-thread.
7. **Escalate disagreements after two rounds** to the user with `to: user, type: question`. Don't argue forever.

---

## Per-Mac install (one-time)

These steps need to happen once per Mac. The Mini was set up first; replicate on the Air.

### 1. SSH key for GitHub

If `ssh -T git@github.com` fails:

```bash
ssh-keygen -t ed25519 -C "crotch.blow@gmail.com"   # accept defaults, optional passphrase
pbcopy < ~/.ssh/id_ed25519.pub
```

Add the key at https://github.com/settings/keys (title it "Mac Mini" or "Mac Air"). Then verify:

```bash
ssh -T git@github.com
# expected: Hi crotchblow-prog! ...
```

### 2. Clone the repo

```bash
git clone git@github.com:crotchblow-prog/agent-bus.git ~/agent-bus
```

### 3. Install the persona

On the **Mini**:

```bash
mkdir -p ~/.claude
cp ~/agent-bus/personas/mira.md ~/.claude/CLAUDE.md
```

On the **Air**:

```bash
mkdir -p ~/.claude
cp ~/agent-bus/personas/aria.md ~/.claude/CLAUDE.md
```

### 4. Install slash commands

Three commands live at `~/.claude/commands/`. On both Macs:

```bash
mkdir -p ~/.claude/commands
```

Then create three files. The exact contents:

**`~/.claude/commands/agent-loop.md`** (start the loop):

```markdown
---
description: Start the agent-bus 60s loop on this Mac (MIRA or ARIA, decided by ~/.claude/CLAUDE.md)
---

/loop 60s Pull the latest from the agent-bus repo: git -C ~/agent-bus pull --rebase. Scan ~/agent-bus/msgs/ for any file with frontmatter to: matching my name (mira or aria — see ~/.claude/CLAUDE.md for which one I am). Skip files I authored. Skip files already in ~/agent-bus/processed/. If there are matching messages: 1. Pick the OLDEST by id timestamp. 2. Read it fully. 3. Decide my response per the persona rules: act on tasks, answer questions, accept/decline/counter-propose, ack, or stop. 4. Write my reply as a NEW file in ~/agent-bus/msgs/ using the protocol filename and frontmatter. 5. Move the message I just handled: git -C ~/agent-bus mv msgs/<file> processed/. 6. git -C ~/agent-bus add . && git -C ~/agent-bus commit -m "<short summary>" && git -C ~/agent-bus push. If there are NO messages addressed to me, idle this tick — do not invent work. Never use cd — always pass paths explicitly (git -C ~/agent-bus ..., absolute paths for file reads/writes). Never edit or delete a message authored by another agent or the user. Never push to branches outside agent-bus from this loop. If two rounds with the peer don't reach agreement, escalate to the user with to: user, type: question and stop responding to that thread.
```

**`~/.claude/commands/agent-stop.md`** (stop the loop):

```markdown
---
description: Stop the agent-bus loop in this Claude Code session
---

/loop stop
```

**`~/.claude/commands/bus-status.md`** (quick status check):

```markdown
---
description: Show agent-bus status — STATUS.md, recent commits, pending messages
---

Show me the current state of the agent-bus. Run these in order and present the results concisely:

1. `git -C ~/agent-bus pull --rebase -q 2>&1 | tail -3`
2. `cat ~/agent-bus/STATUS.md`
3. `git -C ~/agent-bus log --oneline -8`
4. `ls ~/agent-bus/msgs/ 2>/dev/null | grep -v '^_template' || echo "(empty)"`

Format the response as three sections:

**Status** — the STATUS.md content (just the lines under "## Current" — skip the format reference at the top).

**Recent activity** — the git log, one line each.

**Pending in inbox** — the message files (or "(empty)").

Keep the whole response under 20 lines. No commentary, no analysis — just the data.
```

### 5. Install the `delegate` shell function

Append to `~/.zshrc` on each Mac:

```bash
# === agent-bus delegation ===
delegate() {
  if [ "$#" -lt 2 ]; then
    echo "usage: delegate <mira|aria|all> <task description>" >&2
    return 1
  fi
  local to="$1"; shift
  local body="$*"
  case "$to" in mira|aria|all) ;; *) echo "recipient must be mira, aria, or all" >&2; return 1 ;; esac
  local id
  id=$(date -u +%Y-%m-%dT%H-%M-%S)
  local file="$HOME/agent-bus/msgs/${id}-user-to-${to}.md"
  cat > "$file" <<EOF
---
id: $id
from: user
to: $to
type: task
---
$body
EOF
  git -C "$HOME/agent-bus" pull --rebase -q && \
  git -C "$HOME/agent-bus" add "$file" && \
  git -C "$HOME/agent-bus" commit -q -m "user→${to}: ${body:0:60}" && \
  git -C "$HOME/agent-bus" push -q && \
  echo "✓ delegated to ${to}: ${file}"
}
```

Then `source ~/.zshrc` (or open a new terminal).

### 6. Permissions allowlist

To prevent the loop asking permission for every git operation, add these to `~/.claude/settings.local.json` under `permissions.allow`:

```json
"Bash(git -C ~/agent-bus:*)",
"Bash(git -C /Users/<your-user>/agent-bus:*)",
"Read(//Users/<your-user>/agent-bus/**)",
"Write(//Users/<your-user>/agent-bus/**)",
"Edit(//Users/<your-user>/agent-bus/**)"
```

Replace `<your-user>` with `blowcrotch` (Mini) or `chiaoe` (Air).

Claude Code itself can't broaden these rules — the user must add them manually (safety guard).

### 7. Notification sounds (optional but recommended)

Add to `~/.claude/settings.json`:

**Mini**:

```json
{
  "hooks": {
    "Notification": [
      { "matcher": "", "hooks": [{ "type": "command", "command": "afplay /System/Library/Sounds/Submarine.aiff" }] }
    ],
    "Stop": [
      { "matcher": "", "hooks": [{ "type": "command", "command": "test -f /tmp/claude-loop-quiet || afplay /System/Library/Sounds/Purr.aiff" }] }
    ]
  }
}
```

**Air**:

```json
{
  "hooks": {
    "Notification": [
      { "matcher": "", "hooks": [{ "type": "command", "command": "afplay /System/Library/Sounds/Glass.aiff" }] }
    ],
    "Stop": [
      { "matcher": "", "hooks": [{ "type": "command", "command": "test -f /tmp/claude-loop-quiet || afplay /System/Library/Sounds/Tink.aiff" }] }
    ]
  }
}
```

See [Notification sounds](#notification-sounds) for the rationale.

---

## Daily usage

The full interaction surface, once installed:

```
/agent-loop                              # start the loop in this Claude Code session
/agent-stop                              # halt the loop in this session
/bus-status                              # quick read: STATUS, recent log, pending
delegate aria "research charting libs"   # drop a task from any terminal
```

### Typical day

1. **Morning, on the Mini.** Open Claude Code. Type `/agent-loop`. MIRA's loop is now running. Approve any first-time permission prompts (option 2 = "don't ask again for similar"). After ~5 prompts everything is allowlisted.

2. **Open Air, do the same.** Type `/agent-loop`. ARIA is now alive.

3. **Heads-down on EA work, RFP, whatever.** When you think of something to delegate, just type:
   ```
   delegate aria "draft a one-pager landing for Beach & Wne, propose 2 hero copy options"
   ```
   Or from inside any Claude Code session's input box (without leaving your current work):
   ```
   !delegate mira "spike a script that exports MT5 deals to CSV nightly"
   ```

4. **Sound dings.** Submarine = Mini wants you. Glass = Air wants you. Open the right session or run `/bus-status` to see what came in.

5. **End of day.** Loops keep running. Mini stays on. Air loop dies when you close the lid — that's fine, restart in the morning.

### Drafted delegation

For a more complex task, talk to Claude in any session: *"draft a brief for ARIA to do X — show me before sending."* Claude drafts, you tweak, Claude commits to the bus. Same outcome, just refined first.

### Mobile delegation

From iPhone/iPad, GitHub mobile app or web:

1. Open the agent-bus repo.
2. `msgs/` → "Add file" → "Create new file".
3. Filename: `2026-04-25T22-30-00-user-to-mira.md` (with current ISO time).
4. Body:
   ```
   ---
   id: 2026-04-25T22-30-00
   from: user
   to: mira
   type: task
   ---
   Your task here.
   ```
5. Commit. The agent picks it up on next tick.

Working Copy on iPad gives full git control if you prefer.

---

## Notification sounds

| Event | Mini (MIRA) | Air (ARIA) | When it fires |
|---|---|---|---|
| Notification | Submarine | Glass | Claude is waiting on you (permission prompt, question) |
| Stop | Purr | Tink | Claude finished a turn |

### Smart-Stop toggle

The Stop hook checks for `/tmp/claude-loop-quiet`. If the file exists, no sound plays. Use this to silence Stop sounds while loops are noisy:

```bash
touch /tmp/claude-loop-quiet     # silence Stop sounds
rm /tmp/claude-loop-quiet        # restore them
```

Notification (Submarine/Glass) is **always** unaffected — you'll always hear when an agent actually needs you.

### Sound character

- **Submarine** — deep underwater ping (Mini's "down in the engine room" feel)
- **Glass** — bright high clink (Air's "ding!")
- **Purr** — soft warm note (gentle "done" cue, won't fatigue)
- **Tink** — quiet high tap (matches Glass's high register)

Other macOS sounds in `/System/Library/Sounds/` if you want to swap.

---

## Permissions allowlist

Loops do a lot of `git -C ~/agent-bus ...` calls. Without an allowlist, each new command shape (pull, push, add, commit, mv) prompts for permission. The allowlist lives in `~/.claude/settings.local.json`. Pattern:

```
Bash(git -C ~/agent-bus:*)
```

The `:*` is Claude Code's wildcard — matches "any args after this prefix." So one rule covers `git -C ~/agent-bus pull`, `git -C ~/agent-bus add .`, `git -C ~/agent-bus commit -m "..."`, etc.

Claude Code itself **cannot** add allowlist rules — that requires the user to edit `settings.local.json` directly. This is a safety guard.

---

## Custom slash commands

Lives at `~/.claude/commands/<name>.md`. Each file is a markdown prompt template. Typing `/<name>` in a Claude Code session expands the contents into the input.

The three commands here:

| File | Slash | Effect |
|---|---|---|
| `agent-loop.md` | `/agent-loop` | Issues `/loop 60s <full prompt>` to start the bus loop |
| `agent-stop.md` | `/agent-stop` | Issues `/loop stop` |
| `bus-status.md` | `/bus-status` | Has Claude run pull + cat + log + ls and report concisely |

These are **user-level** (`~/.claude/commands/`), so they work in every Claude Code session on the Mac, regardless of project.

### Updating the loop prompt

If you want to change the loop logic (e.g. change interval, add new behaviors), edit `~/.claude/commands/agent-loop.md` and **also** `personas/_loop_prompt.md` in the bus repo so they stay in sync. The `_loop_prompt.md` is documentation only — only the slash command file actually runs.

Existing running loops do **not** pick up changes. You have to `/agent-stop` and `/agent-loop` to apply.

---

## The `delegate` shell function

```bash
delegate <mira|aria|all> <task description>
```

What it does:

1. Generates an ISO-timestamp ID.
2. Writes `~/agent-bus/msgs/<id>-user-to-<recipient>.md` with proper frontmatter and `type: task`.
3. `git -C ~/agent-bus pull --rebase` (avoid push conflicts).
4. `git -C ~/agent-bus add` + `commit` + `push`.
5. Echoes the file path so you can edit it post-hoc if needed.

Run from:

- Any terminal (zsh).
- Any Claude Code session via `!delegate ...` in the input box.

For a longer brief, run `delegate aria "stub"` then open the file and edit:

```bash
$EDITOR ~/agent-bus/msgs/<latest>.md
git -C ~/agent-bus add . && git -C ~/agent-bus commit --amend --no-edit && git -C ~/agent-bus push -f
```

(Force push is OK because nobody else has pulled this commit yet — the loops are on a 60s cadence.)

---

## Loop lifecycle

A `/loop` is a **Claude Code session** with a recurring prompt. It is NOT a daemon, NOT a service. Failure modes:

| Event | Loop survives? |
|---|---|
| You close the terminal window | ❌ dies |
| Mac sleeps | ⏸ pauses, resumes on wake |
| Mac restarts | ❌ dies (and you have to manually restart) |
| `/loop stop` typed | ❌ stops |
| Network drops briefly | ✅ next tick retries |
| Network drops permanently | ❌ effectively dead — git push fails silently |

### Heartbeat check

Quick way to know if a loop is alive:

```bash
delegate aria "heartbeat — reply type: ack with the word pong"
```

If a `pong` reply lands within ≤90s → alive. Otherwise dead.

Or: check `STATUS.md` — both agents update their line each tick. Old timestamp = stale loop.

### Persistence (deferred)

Right now the loops require a Claude Code window to be open. Future improvement: launchd job that auto-starts MIRA's loop after reboot. Not done yet — see [Deferred / future work](#deferred--future-work).

---

## Troubleshooting

### "Loop asks for permission every tick"

Either the allowlist isn't installed (see [Permissions allowlist](#permissions-allowlist)) or the loop was started with the OLD prompt that uses `cd` (which triggers a sandbox warning).

Fix: `/agent-stop`, `git -C ~/agent-bus pull` (in case the canonical prompt updated), `/agent-loop` to restart with the current prompt.

### "I delegated a task but nothing happens"

The recipient's loop isn't running. Check:

- Is there a Claude Code window open on the recipient's Mac?
- Is `/agent-loop` running in it?
- Does `STATUS.md` show a recent timestamp for that agent?

### "Both agents committed at the same time, got a merge conflict"

Each message is a separate file, so conflicts in `msgs/` content are rare. If `STATUS.md` conflicts, resolve manually:

```bash
git -C ~/agent-bus pull --rebase
# resolve in editor
git -C ~/agent-bus rebase --continue
git -C ~/agent-bus push
```

### "I can't tell which agent is dinging"

Sounds map: Submarine = Mini, Glass = Air, Purr = Mini done, Tink = Air done. If they sound the same, double-check `~/.claude/settings.json` on each Mac uses the right sound (the file paths are identical: `/System/Library/Sounds/<Name>.aiff`).

### "GitHub Pages dashboard URL is 404"

The `docs/` folder doesn't exist yet — ARIA is supposed to build it. Check `git log` for any `aria` commits touching `docs/`. If not built yet, the task is in `processed/` waiting on a separate work session.

### "I want to nuke a stuck thread"

Move all related messages to `processed/` manually:

```bash
git -C ~/agent-bus mv msgs/2026-04-25T*-aria-to-mira.md processed/
git -C ~/agent-bus commit -m "manual cleanup: stuck aria↔mira thread"
git -C ~/agent-bus push
```

Then maybe `delegate <agent> "type: stop — drop thread X, start fresh"`.

---

## Deferred / future work

These were discussed but not implemented:

- **CASSI (cloud agent).** A third agent dispatched from the Claude mobile app to a cloud sandbox via the dispatch feature. Would handle one-shot tasks via PRs to this repo. Slot reserved in personas (`tasks/cassi/` folder mentioned in MIRA/ARIA prompts) but agent itself not active.

- **Tailscale "poke" channel.** Push-based wake-up so loops don't have to wait full 60s when there's a new message. Mini and Air would HTTP-poke each other when they push. ~30 lines per side. Skipped for v1; current 60s polling is fine.

- **launchd auto-restart for MIRA.** Mini reboots → MIRA's loop has to be manually restarted. A launchd plist would auto-start it. Not done yet.

- **Bus monitor dashboard.** ARIA's first real task — a static HTML page deployed at `https://crotchblow-prog.github.io/agent-bus/` showing STATUS + recent messages, mobile-first. Direction A approved (`tail -f` style monospace). Awaiting build.

- **Per-repo `AGENTS.md` convention.** When delegating tasks targeting other repos, drop an `AGENTS.md` at the root of each so MIRA/ARIA know conventions, test commands, branch naming. Not yet adopted.

- **Headless / SDK loop.** Replace `/loop` with a small Node/Python daemon using the Claude Agent SDK. No Terminal window required. Most robust path; not built.

---

## TL;DR daily commands

```
/agent-loop          # start the loop on this Mac
delegate aria "..."  # drop a task into the bus
/bus-status          # see what's going on
/agent-stop          # halt the loop
```

Listen for Submarine (Mini wants you) or Glass (Air wants you).

That's the whole system.
