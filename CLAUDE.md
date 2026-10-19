# Improve_Claude — project rules

- Source of truth for the council skill is `claude/council/SKILL.md` in this repo. Never edit `~/.claude/skills/council/SKILL.md` directly — it is overwritten by the git post-commit hook (`.git/hooks/post-commit`) on every commit here.
- Commit after editing the skill — uncommitted edits do not sync to the live copy.
- `copilot/council.prompt.md` is the GitHub Copilot relay-mode port; keep protocol changes mirrored between both files.

## Agent skills

### Issue tracker

GitHub Issues on KaliPoduri/Improve_Claude, via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default labels: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one root `GLOSSARY.md` plus `docs/adr/`. See `docs/agents/domain.md`.
