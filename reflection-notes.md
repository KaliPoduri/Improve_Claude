# Reflection Notes — Claude Code Setup Diagnosis

Date: 2026-07-04
Corpus: 32 sessions across 5 project dirs in `C:\Users\kalip\.claude\projects\` (Connections 6, PlanGenie 13, AI-Projects root 9, FlowTrace 4; current Improve_Claude session excluded). Mined by 6 subagents; clustered and ranked here. Diagnosis only — nothing built.

Session IDs are shortened to first 8 chars. Baseline setup at time of analysis: `bypassPermissions` default, model `claude-fable-5[1m]` + effort `high` saved in settings.json, superpowers/codex/coderabbit/firecrawl plugins, 68 GSD skills + hooks, NotebookLM MCP.

---

## Ranked clusters (most leverage first)

### 1. Cross-AI "council" orchestration is hand-rolled every time — SKILL

**Evidence (6 sessions, 13+ debate rounds, 2 projects):**
- PlanGenie `e77a6905`: 3-round Fable-vs-Codex council via manual SendMessage relays per round.
- PlanGenie `96fbf043` + `1da403c2`: 5-round line-by-line council; 8 codex-rescue launches, 9 polls, 26 edits, all orchestrated turn-by-turn; one TaskOutput timeout dumped 32KB of raw subagent JSONL into main context.
- Connections `73fbd78f`: 3-round council inside /plangenie; 4 user "is it done yet?" polls; hand-written seq-loop pollers + file-mtime liveness checks; un-cancellable zombie Codex job; **2 rounds rerun** because subagents inherited Opus 4.8 after the user's mid-session `/model` switch to Fable 5 didn't apply.
- Connections `24cfed89`: council R1–R2; codex-rescue subagent returned "task forwarded" without results, needed a SendMessage nudge; 1 user interruption during the slow dispatch.
- Connections `9f42a991`: council R3–R5; the same SendMessage nudge was **refused by the codex-rescue subagent as suspected prompt injection**; orchestrator re-derived a `while true` codex-companion.mjs poll loop three separate times.
- The user re-dictates the council spec (round count, agreement rule, model seats) verbatim in `e77a6905`, `96fbf043`, `24cfed89`, `73fbd78f`.

**Recurring sub-failures:** Codex tasks take 8–25 min with zero visibility (8+ manual status polls across sessions, the only user interrupts tied to waiting); codex-rescue's "dispatch-only, don't poll" contract fights the council's need for results — obeyed a nudge once, refused it once (nondeterministic).

**Verdict: build a standalone `/council <file> [rounds]` skill.** It should bake in: (a) Fable seat spawned with explicit `model: 'fable'` (kills the inheritance surprise), (b) Codex seat dispatched via codex-companion.mjs with the background poll loop *inside* the skill, never via SendMessage nudges to codex-rescue, (c) per-round checkpoint to tracking files (survives the 5-hour usage window — user had to manually request this in `24cfed89`), (d) merge-and-commit protocol per round. Note: `plangenie` (own skill), `gsd-plan-review-convergence`, and `gsd-review` all overlap but none was reached for when the Connections HANDOVER-PLAN needed the same workflow — the council wants to be its own composable primitive, not buried in a planning skill. Highest observed time sink in the corpus; medium build cost.

### 2. Session bootstrap/handoff ritual typed by hand at nearly every session start — AUTOMATION (mostly wiring existing pieces)

**Evidence (~13 sessions; 6 are pure-ritual stubs with zero work):**
- `/clear` → `/model` → `/effort` sequence: PlanGenie `16bf8479`, `247b803b`, `e77a6905`, `1d828b32`, `6e31914d`; root `4c13881f`, `5735300a`, `66ace36c`, `88df211e`; Connections stubs `3b7ce2d7`, `990986c4`; FlowTrace `2f2c7e8d`, `05e0e7fc`.
- "read the next_session.md and continue" typed manually: Connections `c9a3b770`, `24cfed89`, `9f42a991`; PlanGenie `56f9582c`.
- "update the tracking files to continue in a fresh new session" typed manually: `73fbd78f`, `6ea687cc`, `24cfed89`.
- Root causes: `/effort ultracode` is session-only by design (must be retyped); `/model` re-run out of distrust after the Opus-vs-Fable council rerun in `73fbd78f`; two sessions (`3b7ce2d7`, `990986c4`) exist *only* to run `/clear` + `/model`.

**Verdict: automation, low build cost, high frequency.**
- Add a SessionStart hook: if `next_session.md` exists in cwd, inject its content + `git status --short` into context. Kills the most-typed prompt in the corpus.
- Stop re-running `/model` and `/effort`: Fable 5 [1m] and effort high are already the saved defaults in settings.json — the ritual is redundant except for ultracode, which cannot persist (harness design). Worth one line in CLAUDE.md so future sessions don't re-litigate it.
- The handoff side already exists as `gsd-pause-work` / `gsd-resume-work` — installed, never invoked in any mined session. Either adopt them or wrap the exact next_session.md protocol in a 10-line `/handoff` skill.

### 3. `/codex:cancel` is broken on this machine (MSYS path mangling) — FIX

**Evidence (3 failures, 2 projects):**
- FlowTrace `e64db429`: cancel attempted twice against a 7-min review; both failed with `taskkill /PID 10076 /T /F: exit=1: ERROR: Invalid argument/option - 'C:/Program Files/Git/PID'` — Git Bash rewrote the `/PID` flag into a filesystem path. Review could not be aborted.
- Connections `73fbd78f`: same class of failure plus `No job found for "task-mr67qwbw-3e37v5"` — left a zombie job displayed as running for the rest of the session.

**Verdict: fix.** Patch locally (double-slash `//PID` or `MSYS_NO_PATHCONV=1` in the plugin's cancel path in `codex-companion.mjs`) and/or report upstream to openai/codex-plugin-cc. Tiny cost; without it every mistaken 8–25 min Codex launch must be waited out — which is exactly what caused the manual-polling anxiety in cluster 1.

### 4. Duplicate expensive work and lost prompts from session churn — FIX (hygiene rule)

**Evidence (4 incidents):**
- FlowTrace `e64db429` → `09859035`: identical Codex review run to completion **twice in 20 minutes** (>12 min combined Codex time, same 2 findings) because session 1's findings were displayed but abandoned during a plugin-reload restart.
- Root `66ace36c`: full typed project brief swallowed by `Unknown command: /plangenie` (project-scoped skill not visible); retyped in `d7f20a9e`.
- PlanGenie `247b803b` → `b9f52428`: the same 1.7KB founding prompt pasted twice in 3 minutes — first session abandoned mid-task for a plugin update/reload.

**Verdict: fix, not a skill.** Two CLAUDE.md lines cover it: (a) before abandoning/restarting a session for a reload, first apply or file pending findings and save any long draft prompt to a file; (b) after installing a new skill/plugin, verify the command registers (`/reload-plugins`, then check) before composing a long invocation. The harness constraint (skills not hot-loaded) can't be built away.

### 5. Skill two-copy drift (project file vs `~/.claude/skills/` copy) — SMALL AUTOMATION

**Evidence:** PlanGenie `6e31914d` — an entire session existed only to hand-sync the council-edited `PLANGENIE.md` into `~/.claude/skills/plangenie/` (user-level copy had silently gone stale, 7482 vs 9395 bytes). The GitHub repo is a third copy. Will recur on every future edit; `56f9582c` shows the global install that created the fork.

**Verdict: automate cheaply.** Either an NTFS junction/hardlink so there's one physical file, or a 3-line sync step. Not worth a skill; worth 10 minutes.

### 6. `gh` CLI unauthenticated — FIX (one command)

**Evidence (2 sessions, 2 projects):** Connections `73fbd78f` (`gh` exit 4 → fallback to raw git + credential manager during first push) and PlanGenie `56f9582c` (same error, same workaround). Latent: every future PR/issue/API call re-hits it.

**Verdict: run `gh auth login` once.** Best cost-benefit ratio in this list.

### 7. Hook overhead on every stop/edit — FIX (conscious trim)

**Evidence:** codex `stop-review-gate-hook.mjs` fired on ~50+ stops across all mined July/June sessions (0.5–3s each), **never once blocked or contributed** — including pure Q&A and config sessions (root `5c545c60`, FlowTrace sessions where the gate was disabled yet the hook still ran). It was loop-enabled across all 8 repos in root `9d40fd4e`. Separately, 6 GSD hooks fire on every Write/Edit/Bash/Read even in non-GSD projects (all mined sessions ran outside GSD phase structure).

**Verdict: decide deliberately.** If stop-time Codex review isn't wanted, disable the gate (`/codex:setup`); if GSD isn't used in a project, the hook stack is pure latency there. Small constant tax, trivial to remove.

### 8. Windows environment gaps — MINOR FIXES

- **Named agents need tmux/WSL:** `73fbd78f` hit `To use agent swarms, you need tmux...` twice → user interrupt + pasted external advice; workaround (unnamed agents) is recorded in progress.md. Also probed in `1d828b32`. Verdict: keep the workaround documented in global CLAUDE.md (one line), or set `teammateMode` away from tmux; low frequency now that it's known.
- **`pdftoppm` missing** blocked PDF reading in `b9f52428` (advice given was Mac/Linux-only); contributed to 2 interrupts. Verdict: `choco install poppler` once if PDF parsing matters.
- **CRLF warnings** on tracking files (30+ occurrences, FlowTrace + PlanGenie): cosmetic; a `.gitattributes` or `core.autocrlf` setting silences it. Optional.

### 9. Things that recur but want NOTHING built

- **Monorepo-as-folder mixup** (`1da403c2`, `9d40fd4e`: F:\AI_Projects had a `.git` capturing 229 commits of all projects; caused subtree splits, ~17 corrective commands, and the corpus's scariest moment — an interrupt during destructive cleanup). Already resolved in-session (`.git` removed, per-folder repos). Worth a one-time verification that every project folder is now its own repo; nothing to build.
- **Subagent model inheritance surprise** (`73fbd78f`, cost 2 council reruns): now understood, and the fix (explicit `model:` param) belongs inside the cluster-1 skill. progress.md already records it.
- **Context bloat mechanics** (`6ea687cc`: 173KB PDF triplicated ≈ 700KB of a 911KB transcript; `1da403c2`: 32KB TaskOutput dump): harness behavior. Behavioral guidance only — delegate large-document parsing to a subagent so the content stays out of main context.
- **Home/work split** (Connections: all M0 gates need the office environment; user couriers M0-GATES.md/HANDOVER-PLAN.md to an org Copilot agent; "no. i am at home" correction in `24cfed89`): structural, not tooling. At most, note current location in next_session.md so resumes don't assume the wrong machine.
- **/codex:review `--effort` flag trap** (FlowTrace `09859035`, one failed background launch): already codified in global CLAUDE.md the same day. Done.

---

## What's already working (don't touch)

- **next_session.md handoff discipline**: across the Connections 3-session day, zero re-explaining of project substance; resumes were one line. The protocol works — cluster 2 just removes the typing.
- **bypassPermissions**: zero permission-friction events in the whole corpus.
- **AskUserQuestion decision elicitation**: 12-question M0 sweep in `24cfed89` and the plangenie interview both landed well; the only misfires were a question asked after the answer was delivered (`1d828b32`) and canned options missing reality (`1da403c2`).
- **superpowers plan/execute skills** (`56f9582c`) and **CLAUDE.md learning-capture** (`09859035`) both did their jobs.

## Summary table

| # | Cluster | Sessions | Verdict |
|---|---------|----------|---------|
| 1 | Hand-rolled Fable+Codex council | 6 (2 projects) | New `/council` skill |
| 2 | Start/handoff ritual typed each session | ~13 | SessionStart hook + adopt gsd-pause/resume |
| 3 | /codex:cancel broken (MSYS taskkill) | 3 failures | Patch / report upstream |
| 4 | Duplicate work + lost prompts on restart | 4 incidents | 2 CLAUDE.md hygiene lines |
| 5 | Skill copy drift (PLANGENIE.md x3 copies) | 1, will recur | Junction or sync step |
| 6 | gh unauthenticated | 2 | `gh auth login` once |
| 7 | Stop-gate + GSD hook overhead everywhere | all sessions | Disable where unused |
| 8 | Windows gaps (tmux, poppler, CRLF) | scattered | One-line fixes, optional |
| 9 | Monorepo, model-inheritance, context bloat, home/work | — | Nothing to build |
