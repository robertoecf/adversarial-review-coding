# claude-code-review-subagents

Review triad plugin for Claude Code: prompt optimization, adversarial red-team review, plan validation, and an orchestrator that routes reviews intelligently.

## Skills

| Skill | Command | What it does |
|-------|---------|--------------|
| **Prompt Optimize** | `/prompt-optimize` | Analyze and optimize prompts, system instructions, skill definitions |
| **Adversarial Review** | `/adversarial-review` | Red-team review: security holes, failure modes, race conditions, cost traps |
| **Plan Review** | `/plan-review` | Validate implementation plans before execution |
| **Codex Review** | `/codex-review` | Cross-model review via Codex CLI (GPT-5.4) — independent second opinion + disagreement analysis |
| **Review All** | `/review-all` | Orchestrator — classifies input and routes to the right reviewer(s) |

## Install

```bash
# Add as a marketplace source, then install
claude plugin marketplace add robertoecf/claude-code-review-subagents
claude plugin install claude-code-review-subagents
```

Or from a local clone:

```bash
git clone https://github.com/robertoecf/claude-code-review-subagents.git
claude plugin marketplace add /path/to/claude-code-review-subagents
claude plugin install claude-code-review-subagents
```

## Usage

### Prompt Optimization

```
/prompt-optimize
# Then paste your system prompt, agent instruction, or SKILL.md
```

Modes: **Critique** (issues only), **Optimize** (rewrite + diff), **Compare** (side-by-side scoring of two prompts).

### Adversarial Review

```
/adversarial-review
# Then paste code, config, or point to files
```

Modes: **Security** (OWASP, injection, auth), **Robustness** (race conditions, failure cascades), **Cost** (scaling, token budgets), **Full Red Team** (all three).

### Plan Review

```
/plan-review
# Then paste your implementation plan
```

Modes: **Validate** (full review), **Feasibility** (Codex-powered codebase check), **Quick Check** (lightweight for short plans).

### Codex Review (Cross-Model)

```
/codex-review
# Sends input to both Claude and Codex CLI (GPT-5.4) for independent analysis
```

Modes: **Code Review** (security + robustness), **Git Review** (`--uncommitted` or `--base`), **Prompt Review**, **Plan Review**, **Custom Query**.

Requires [Codex CLI](https://github.com/openai/codex) with `codex login` completed. Gracefully falls back to Claude-only if unavailable.

### Review All (Orchestrator)

```
/review-all
# Paste any content — the orchestrator classifies and routes it
```

Automatically detects whether your input is a prompt, plan, code, or mixed content and invokes the appropriate reviewer(s).

## Output Format

All skills use a consistent severity scale and finding format:

- **P0** (Critical) — must fix before merge/deploy
- **P1** (High) — should fix in this iteration
- **P2** (Medium) — fix when convenient
- **P3** (Low) — optional improvement

Each finding includes evidence, impact assessment, mitigation options, and a specific recommendation.

## Codex CLI Integration (Optional)

Skills can optionally use [Codex CLI](https://github.com/openai/codex) for cross-model validation. When available, it provides a second opinion on findings. When unavailable, all analysis runs within Claude only — no degradation.

## Design Principles

- **Self-contained skills** — each SKILL.md has everything needed, no external dependencies
- **Review-only** — skills analyze and recommend but never modify your code
- **Honest reviews** — clean code gets a clean report, no manufactured findings
- **Token-disciplined** — hard caps on output length per skill
- **Concrete examples** — every skill includes input→output demonstrations

## License

MIT
