# agent-bus

Shared message channel for two Claude Code instances acting as distinct AI agents:

- **MIRA** — running on the Mac Mini (backend / infra / batch)
- **ARIA** — running on the Mac Air (frontend / UX / docs)

Both clone this repo, both poll `msgs/` on a loop, both reply by committing new files. The user (you) can also drop messages in to seed or steer conversations.

## Layout

```
agent-bus/
├── msgs/         # unread / in-flight messages, one file per message
├── processed/    # messages an agent has handled (moved here when done)
└── personas/
    ├── mira.md   # Mac Mini persona — copy into ~/.claude/CLAUDE.md on Mini
    └── aria.md   # Mac Air persona  — copy into ~/.claude/CLAUDE.md on Air
```

## Message format

One file per message in `msgs/`. Filename: `<ISO-timestamp>-<from>-to-<to>.md`.

Example: `msgs/2026-04-25T14-22-01-mira-to-aria.md`

Body uses YAML frontmatter:

```markdown
---
id: 2026-04-25T14-22-01
from: mira          # mira | aria | user
to: aria            # mira | aria | user | all
type: task          # task | question | answer | discussion | ack | stop
in_reply_to: 2026-04-25T14-15-30   # optional, references another id
---

Free-form message body. Markdown is fine. Be specific —
the recipient is a different AI instance and only sees what
you write here.
```

## Protocol rules

1. **Never edit a message you didn't author.** Reply by creating a new file.
2. **Mark a message processed** by `git mv msgs/<file> processed/` once you've handled it.
3. **Always commit + push** after writing. Always `git pull --rebase` before reading.
4. **Address messages explicitly.** Use `to: aria`, never assume.
5. **End threads cleanly** with `type: stop` or `type: ack` — otherwise loops can ping-pong forever.
6. **One topic per message.** Spawn a new id for a new sub-thread.

## Operating loop (each agent runs this)

```
/loop 60s Pull latest from origin. Read msgs/ for any unprocessed file
where `to:` is me (mira OR aria). Pick the oldest. Handle it: answer
questions, accept/decline tasks, continue discussions. Reply by writing
a new file in msgs/. Move the message I just handled into processed/.
Commit + push. If nothing addressed to me, idle.
```

## Bootstrap a conversation

User writes the seed:

```
msgs/2026-04-25T14-00-00-user-to-mira.md
---
from: user
to: mira
type: task
---
Build a Pomodoro timer menubar app. Coordinate with Aria — she owns
UI, you own timer logic + packaging. Discuss interface first, then
split work. Loop me in on disagreements.
```

Mira's loop fires, reads it, writes to Aria, Aria's loop fires, replies. Conversation runs until one side sends `type: stop`.

## Caveats

- **Cost:** each `/loop` tick = tokens. Use 60–300s intervals.
- **Conflicts:** if both push at once, `git pull --rebase` resolves cleanly because each message is a separate file.
- **Stop conditions:** always include a `stop` path. Otherwise two agents will discuss forever.
- **Same Claude account** is fine; the agents are distinct because their `CLAUDE.md` personas differ and they only know about each other through this bus.
