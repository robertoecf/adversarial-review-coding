# Codex CLI Integration Reference

## Environment

- **Binary**: `/opt/homebrew/bin/codex` (v0.114.0+)
- **Auth**: `~/.codex/auth.json`
- **Config**: `~/.codex/config.toml`
- **Default model**: GPT-5.4, reasoning effort: high

## Detection

```bash
which codex && test -f ~/.codex/auth.json && echo "CODEX_AVAILABLE" || echo "CODEX_UNAVAILABLE"
```

Both conditions must pass. If either fails, skip Codex entirely.

## Invocation Patterns

### Non-interactive exec (preferred)
```bash
codex exec --full-auto --ephemeral -o /tmp/codex-review-output.md "<prompt>"
```

### Git review (built-in)
```bash
codex review --uncommitted                  # staged + unstaged + untracked
codex review --base main                    # diff against branch
codex review --commit <sha>                 # specific commit
```

### Key flags
| Flag | Purpose |
|------|---------|
| `--full-auto` | No approval prompts, sandboxed workspace-write |
| `--ephemeral` | Don't persist session files |
| `-o <file>` | Write final agent message to file |
| `-m <model>` | Override model (e.g., `-m o3`) |
| `-C <dir>` | Set working directory |
| `--json` | JSONL event stream output |
| `-c model_reasoning_effort="high"` | Override reasoning effort |

## When to Use

- Cross-model validation of security/robustness findings
- Git diff reviews via `codex review`
- Codebase feasibility checks
- Any review where two model families seeing the same input adds confidence

## When NOT to Use

- Simple reviews with obvious findings
- When auth is missing (detection fails)
- Extremely large inputs (>2000 lines) without chunking

## Graceful Degradation

If Codex is unavailable, all analysis runs within Claude only. No skill
should fail or degrade quality because Codex is missing. Codex findings
are additive, never required.

## Timeout

Always wrap with `timeout 120` for non-interactive calls:
```bash
timeout 120 codex exec --full-auto --ephemeral -o /tmp/codex-review-output.md "<prompt>"
```

## Cleanup

Remove temp files after every Codex invocation:
```bash
rm -f /tmp/codex-review-input.txt /tmp/codex-review-output.md
```

## Disagreement Handling

When Claude and Codex disagree:
1. Flag the disagreement explicitly in output
2. Present both perspectives with reasoning
3. Take the more conservative (higher severity) assessment
4. Tag with `[cross-model disagreement]`
