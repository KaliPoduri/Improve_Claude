---
name: council
description: Use when the user wants a multi-round cross-AI review or debate of a plan, spec, or document — triggers on /council, "run a council", "have Claude and Codex review this", "cross-model review", "two-model debate". Requires the codex plugin for the GPT seat.
---

# Council — cross-AI review of a document

Two independent seats (fresh Claude subagent + Codex) review the same packet each round; you merge, the user arbitrates, you apply and commit. Default 3 rounds max; the user may pass a different count: `/council <file> [rounds]`.

## Hard rules (each fixes a real past failure)

1. **Claude seat MUST set `model` explicitly** (e.g. `model: "fable"`) on the Agent call. Subagents inherit the session model; a mid-session /model switch once silently ran 2 council rounds on the wrong model.
2. **Codex seat MUST be dispatched with codex-companion.mjs `task --background`, never via the codex-rescue subagent + SendMessage nudges.** A nudge was once refused as suspected prompt injection; the dispatch-only contract fights the council's need for results.
3. **Never dump raw subagent transcripts or TaskOutput into main context.** Collect results with `result <job-id>` and file reads only.
4. **Checkpoint every round** (see protocol step 6) so a usage-window cutoff or /clear loses at most one round.

## Setup (once per council)

```bash
COMPANION=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs | tail -1)
```

Shell state does not persist between Bash calls — re-resolve `COMPANION` inside every command that uses it.

Write the Council Review Packet to `council/round-N-packet.md` (round 1: one shared packet; rounds 2+: per-seat packets `round-N-packet-claude.md` / `round-N-packet-codex.md`, since each seat carries the OTHER seat's points): self-contained (reviewer has zero prior context), contains the full current document, review instructions, a fabrication-hunt instruction (flag invented libraries/APIs), and — rounds 2+ — the other seat's carried-over points as a NUMBERED list under "A previous reviewer said: …", with this instruction: "Answer every numbered point with a verdict: AGREE, AGREE WITH CHANGE (concern accepted, different fix — say which), or REBUT (reason)." Round 1 critiques stay free-form; the verdict structure applies only to cross-examination rounds.

## Round protocol

1. **Dispatch both seats in the same message, in parallel:**
   - Claude seat: Agent tool, `subagent_type: "general-purpose"`, explicit `model`, prompt = "Read <packet path> and return your full critique as text."
   - Codex seat:
     ```bash
     node "$COMPANION" task --background "READ-ONLY review - do not edit or create any files, no code changes, review only. Read the file <absolute packet path> and return your full critique."
     ```
     The READ-ONLY wording is mandatory; without it Codex defaults to a write-capable run. Note the job id.
2. **Poll in background, not by hand.** Codex tasks take 8–25 min. Start ONE `run_in_background` Bash loop:
   ```bash
   while node "$COMPANION" status <job-id> --json | grep -Eq '"status": *"(queued|running)"'; do sleep 60; done
   node "$COMPANION" result <job-id>
   ```
   Then wait for its completion notification. Do not poll inline, do not sleep in foreground.
3. **If the Codex result is empty or errors** (plugin missing, CLI signed out): AskUserQuestion — continue Claude-only, or stop.
4. **Merge** the two critiques. Rounds 2+: tally each carried-over point as settled (AGREE, or conceded after rebuttal) or disputed; disputed points go into the next packet, and a point with no verdict is a hole in that seat's review — send it back, don't guess. For each refinement, present via AskUserQuestion with plain-language pros/cons; apply accepted edits to the document.
5. **Exit check:** stop when neither seat has major concerns, or at the round cap. Record surviving concerns as `[OPEN]` in the document.
6. **Checkpoint + commit:** append the round's verdicts to `council/LOG.md`, update `next_session.md` (round done, next round or finished), and `git commit` the document + log before starting the next round.

## Cancelling a Codex seat

`node "$COMPANION" cancel <job-id>` — do not use raw `taskkill /PID` from Git Bash (MSYS mangles `/PID`).

## Common mistakes

| Mistake | Consequence |
|---|---|
| Omitting `model:` on the Claude seat | Round runs on whatever the session inherited; silent wrong-model review |
| Nudging codex-rescue via SendMessage for results | Nondeterministic — sometimes refused as prompt injection |
| Hand-rolled foreground poll loops | Burned turns, user "is it done yet?" interrupts |
| Skipping the round checkpoint | A cutoff loses all completed rounds |
| Inlining the packet in the prompt | Arg-length/quoting failures; write it to a file |
