# Next Session — Improve_Claude

**Task:** Implemented reflection-notes.md suggestions (2026-07-04). All built items done and verified.

**Done:**
- /codex:cancel MSYS bug patched in plugin cache v1.0.5 process.mjs (MSYS_NO_PATHCONV on taskkill). Re-patch after plugin updates; worth reporting upstream to openai/codex-plugin-cc.
- SessionStart hook `~/.claude/hooks/resume-next-session.js` auto-injects next_session.md + git status on startup/clear.
- Global CLAUDE.md: "This Machine" section (defaults, restart hygiene, tmux, large-doc delegation).
- PLANGENIE.md drift: git post-commit hook in PlanGenie repo syncs to ~/.claude/skills/plangenie/. Hardlinks ruled out (Edit tool breaks them — tested).
- /council skill at ~/.claude/skills/council/SKILL.md — Codex dispatch/poll/result verified live; subagent comprehension test passed.
- Verified: no root .git in F:\AI_Projects; per-project repos intact.

**Open (user actions):**
1. Run `gh auth login` (still unauthenticated).
2. Decide hook overhead: GSD hooks fire in all projects; codex stop-gate already disabled (stopReviewGate=false) but its hook still runs on Stop.
3. Optional: `choco install poppler` (PDF), .gitattributes for CRLF noise.
