# CLAUDE.md — Claude Code Directives

## Plugin: adversarial-review-coding v0.3.0

Adversarial review plugin for coding workflows. Spawns background subagents
that call external models (Codex CLI, Gemini CLI) for independent red-team
analysis, cross-validates against Claude, and returns unified findings.

## Architecture

- All skills use `model: inherit` (main session model, typically opus)
- Adversarial skills spawn `Agent(run_in_background=true)` with fresh context
- Subagent calls Codex/Gemini CLI, cross-validates, returns only final output
- Main session is never blocked during review

## Critical Gotchas

- `model: haiku` in skill frontmatter inherits FULL conversation context — blows 200k limit in long sessions. Use `model: inherit` + `Agent(model=haiku)` if you need haiku with fresh context.
- `"skills": "./skills/"` is REQUIRED in plugin.json for Claude Code to discover SKILL.md files
- Plugin cache lives at `~/.claude/plugins/cache/` — must be manually updated after source changes during dev
- Global gitignore at `~/.config/git/ignore` blocks `.claude/settings.local.json` — use `git add -f` to include it
- Codex CLI (GPT-5.4 reasoning:high) needs 300s timeout, not 120s
- Gemini CLI: `-p` for non-interactive, `-y` for auto-accept. Model cascade: 3.1-pro → 3.1-flash-lite → 2.5-pro → 2.5-flash

## Tool Preferences

- Use `Read` over `cat` / `head` / `tail`
- Use `Grep` over `grep` / `rg`
- Use `Glob` over `find` / `ls`

## Available Skills

- `/adversarial-plan-review` — background agent: Codex/Gemini critique → cross-validate → revised plan
- `/coding-adversarial-review` — background agent: Codex/Gemini red-team → cross-validate → critics
- `/prompt-optimize` — prompt analysis and optimization (runs on main session model)
- `/review-all` — router that classifies input and suggests the right skill

## Fallback Chain (single-model rule)

Call ONLY the first available. Stop at first success.
1. Codex CLI (GPT-5.4) — `codex exec --full-auto --ephemeral` (timeout 300s)
2. Gemini cascade: `gemini-3.1-pro-preview` → `3.1-flash-lite-preview` → `2.5-pro` → `2.5-flash` (timeout 180s)
3. Claude-only (last resort)

## Plugin Dev Workflow

- Edit source at `/Users/mac/Documents/Repos/claude-plugins/adversarial-review-coding/`
- Copy to cache: `cp <src>/skills/*/SKILL.md ~/.claude/plugins/cache/adversarial-review-coding/adversarial-review-coding/0.3.0/skills/*/`
- Reload: `/reload-plugins`
- Skill invocation requires fully qualified name: `adversarial-review-coding:coding-adversarial-review`
- Install via marketplace: `claude plugin marketplace add <path>` then `claude plugin install <name>`

## File Size Discipline

No single file in this plugin should exceed 500 lines.
