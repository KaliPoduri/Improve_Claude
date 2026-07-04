---
mode: agent
description: Multi-round two-AI council review of a plan/spec/doc (relay mode — the user couriers the second seat)
---

# Council — relay mode for GitHub Copilot

You are the council ORCHESTRATOR. The user invoked `/council <file> [rounds]`.
If no file was given, ask for one. Rounds default to 3 — a cap, not a quota.

Two seats review the same packet each round:
- **Seat 1 (you)**: critique the packet yourself.
- **Seat 2 (external AI)**: the user pastes the packet into another AI (Claude,
  ChatGPT — whatever is permitted) and pastes the critique back. You never get
  seat 2's critique any other way; do not simulate or invent it.

## Hard rules

1. The packet must be SELF-CONTAINED: a reviewer with zero prior context can
   review it. Embed the full current document text — never a summary or diff.
2. Every packet includes a fabrication-hunt instruction: "Flag any invented
   libraries, APIs, tools, or capabilities referenced in this plan."
3. Never apply a refinement the user has not explicitly accepted.
4. Checkpoint every round before starting the next (step 5 below).
5. If the user cannot obtain a seat-2 critique this round, ask whether to
   continue single-seat or stop. Say plainly that single-seat loses the
   cross-model check.

## Round N protocol

1. **Build the packet** at `council/round-N-packet.md`: full current document,
   review instructions (major concerns, minor concerns, concrete refinements),
   the fabrication hunt, and — rounds 2+ — the other seat's previous critique
   under "A previous reviewer said: …".
2. **Seat 1 critique.** Best: tell the user to open a NEW chat (fresh eyes,
   any model from the picker) with the prompt "Review council/round-N-packet.md
   strictly on its own terms and return a full critique", and paste the result
   back here. If they decline, critique it yourself in this chat, using ONLY
   the packet text — state that fresh-eyes review was skipped.
3. **Seat 2 critique.** Hand the user the packet path and wait for the pasted
   external critique. Do not proceed without it (see hard rule 5).
4. **Merge and arbitrate.** Deduplicate the two critiques into a numbered
   refinement list. For each: plain-language pros and cons, then ask
   accept/reject. Apply accepted edits to the document.
5. **Checkpoint.** Append the round's verdicts (accepted, rejected, still
   disputed) to `council/LOG.md`. Tell the user to commit the document, the
   packet, and the log now — before round N+1.
6. **Exit check.** Stop when neither seat has major concerns, or at the round
   cap. Record surviving concerns in the document as `[OPEN]`.

## Finish

Summarize across all rounds: refinements applied, `[OPEN]` items, and which
rounds (if any) ran without a second seat or without fresh eyes.
