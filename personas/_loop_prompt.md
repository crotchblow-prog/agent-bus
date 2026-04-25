# Loop prompt for both Macs

This is the recurring prompt each agent runs via `/loop`. Same text on both Macs — the persona in `~/.claude/CLAUDE.md` tells the agent which name they answer to.

## How to start the loop

On the Mac Mini (after copying `mira.md` to `~/.claude/CLAUDE.md`):

```
/loop 60s <paste prompt below>
```

On the Mac Air (after copying `aria.md` to `~/.claude/CLAUDE.md`):

```
/loop 60s <paste prompt below>
```

Adjust the interval. 60s is a sane default. 30s is responsive, 300s is cheap.

## The prompt

```
Pull the latest from the agent-bus repo (cd ~/agent-bus && git pull --rebase).

Scan msgs/ for any file with frontmatter `to:` matching my name (mira or aria — see ~/.claude/CLAUDE.md for which one I am). Skip files I authored. Skip files already in processed/.

If there are matching messages:
1. Pick the OLDEST by id timestamp.
2. Read it fully.
3. Decide my response per the persona rules: act on tasks, answer questions, accept/decline/counter-propose, ack, or stop.
4. Write my reply as a NEW file in msgs/ using the protocol filename and frontmatter.
5. `git mv` the message I just handled into processed/.
6. `git add . && git commit -m "<short summary>" && git push`.

If there are NO messages addressed to me, idle this tick — do not invent work.

Never edit or delete a message authored by another agent or the user.
Never push to branches outside agent-bus from this loop.
If two rounds with the peer don't reach agreement, escalate to the user with `to: user, type: question` and stop responding to that thread.
```

## Stopping the loop

In each Claude Code session, type `/loop stop` or interrupt with Esc and tell the agent to stop the loop.

## When to manually break in

You can always send a normal message in the same session — the loop pauses while you talk. To inject a high-priority instruction into the bus from anywhere, just commit a `from: user, to: mira|aria|all` file in `msgs/` and the next tick picks it up.
