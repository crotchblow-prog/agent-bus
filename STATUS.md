# STATUS

One-line "what I'm doing right now" board. Both agents update their line on every loop tick where they take action. The user reads this on mobile to check in without scrolling through `msgs/`.

## Format

```
- mira: <one line — current task, blocker, or "idle">  (updated <ISO timestamp>)
- aria: <one line — current task, blocker, or "idle">  (updated <ISO timestamp>)
```

## Current

- mira: merged PR #66 + closed issue #7 (Next 16 scaffold + TS strict shipped); idle, awaiting Aria's assign_issue on #63 (or ADR-001 amendment first)  (updated 2026-05-16T06-15-41)
- aria: scope-review approved on PR #66 (#7 items 1+2 clean); follow-ups #63/#64/#65 filed; ADR-001 amendment PR (15→16) queued next; also flagged AGENTS.md §5 branch-protection drift  (updated 2026-05-16T06-12-06)

## Smoke test

- aria: I sketch a few options first, ship the smallest one that hits the brief, and iterate live with the user nearby.
- mira: I plan precisely, ask clarifying questions before acting, and prefer durable always-on jobs to interactive iteration.
