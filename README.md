# adversarial-review-coding

## Adversarial Review with LLMs

Adversarial review is the practice of submitting work to an independent model for red-team analysis, then cross-validating the findings against your primary model's own assessment. Two models examining the same artifact from different angles catch more issues than either alone — each has different training biases, blind spots, and reasoning patterns. Disagreements between models surface the highest-value findings: the ones a single reviewer would miss.

This plugin applies adversarial review to **coding workflows** inside [Claude Code](https://docs.anthropic.com/en/docs/claude-code). It spawns background subagents that call external models (Codex CLI / Gemini CLI), cross-validate against Claude's analysis, and return unified findings — without blocking your main session. The result: security vulnerabilities, logic errors, and architectural gaps caught earlier, with higher confidence, and with clear severity ratings.

## How It Works

```
Main Session (Opus) ─── continues working, not blocked
       │
       └─► spawns background Agent
                │
                ├─ 1. Send artifact to external model (Codex/Gemini)
                ├─ 2. Run independent Claude analysis
                ├─ 3. Cross-validate both sets of findings
                │     ├─ [cross-validated] = high confidence
                │     ├─ [external-only]  = flagged for review
                │     └─ [claude-only]    = flagged for review
                └─ 4. Return unified output (P0-P3 severity)
```

## Skills

| Skill | What it does |
|-------|-------------|
| `/coding-adversarial-review` | Red-team code, configs, and diffs. Returns security/robustness critics with exploit chains. |
| `/adversarial-plan-review` | Critique implementation plans. Returns a revised plan with risk analysis. |
| `/prompt-optimize` | Analyze and optimize prompts for clarity, token efficiency, and conflicts. Runs inline (no external model). |
| `/review-all` | Lightweight router — classifies your input (code, plan, or prompt) and routes to the correct skill. |

## External Model Fallback Chain

The plugin calls **one** external model per review (first available wins):

```
1. Codex CLI (GPT-5.4)    ◄── primary
2. Gemini CLI              ◄── fallback (3.1-pro → 3.1-flash-lite → 2.5-pro → 2.5-flash)
3. Claude-only analysis    ◄── last resort
```

## Install

```bash
claude plugin marketplace add robertoecf/adversarial-review-coding
claude plugin install adversarial-review-coding
```

### Prerequisites

At least one external model CLI must be authenticated:

```bash
# Primary: Codex CLI (GPT-5.4)
codex login

# Fallback: Gemini CLI
gemini auth login
```

## Usage

```bash
/coding-adversarial-review          # paste code, point to files, or "review uncommitted"
/adversarial-plan-review            # paste your implementation plan
/prompt-optimize                    # paste a system prompt or skill definition
/review-all                         # paste anything — auto-routes to the right skill
```

## Architecture Diagram

[![Architecture](https://excalidraw.com/og/5S7Pzstx9npv10zCbtwkV)](https://excalidraw.com/#json=5S7Pzstx9npv10zCbtwkV,AJzXgdXVwoc3ojldcMgGZA)

> [Open full diagram in Excalidraw](https://excalidraw.com/#json=5S7Pzstx9npv10zCbtwkV,AJzXgdXVwoc3ojldcMgGZA)

## License

MIT
