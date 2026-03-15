# Cross-Model Fallback Chain

## CRITICAL RULE

Call ONLY the FIRST available model. Do NOT call multiple models.
Stop as soon as one succeeds.

## Chain Order

```
1. Codex CLI (GPT-5.4)          ◄── try first
   ↓ if unavailable
2. Gemini CLI (model cascade)   ◄── try second
   ↓ if unavailable
3. Claude-only                   ◄── last resort
```

## Model 1: Codex CLI

### Detection
```bash
which codex && test -f ~/.codex/auth.json && echo "CODEX" || echo "NO_CODEX"
```

### Invocation
```bash
cat << 'TEMPLATE_EOF' > /tmp/cross-model-input.txt
<filled template>
TEMPLATE_EOF
timeout 120 codex exec --full-auto --ephemeral -o /tmp/cross-model-output.md "$(cat /tmp/cross-model-input.txt)"
cat /tmp/cross-model-output.md
```

### Git Review (code review only)
```bash
codex review --uncommitted 2>&1 | tee /tmp/cross-model-output.md
```

**If Codex succeeds → STOP. Do not call Gemini.**

## Model 2: Gemini CLI (cascade by capability)

Only reached if Codex is unavailable or fails.

### Detection
```bash
which gemini && test -f ~/.gemini/oauth_creds.json && echo "GEMINI" || echo "NO_GEMINI"
```

### Model Cascade (try in order, stop at first success)

| Priority | Model ID | Tier |
|----------|----------|------|
| 1 | `gemini-3.1-pro-preview` | 3.1 family best |
| 2 | `gemini-3.1-flash-lite-preview` | 3.1 family lite |
| 3 | `gemini-2.5-pro` | 2.5 family best |
| 4 | `gemini-2.5-flash` | 2.5 family lite |

### Invocation
```bash
cat /tmp/cross-model-input.txt | timeout 120 gemini -p "" -y -m gemini-3.1-pro-preview > /tmp/cross-model-output.md 2>/dev/null
```

If the command fails or output is empty, try the next model in order:
1. `gemini-3.1-pro-preview`
2. `gemini-3.1-flash-lite-preview`
3. `gemini-2.5-pro`
4. `gemini-2.5-flash`

Stop at the first that succeeds.

**If any Gemini model succeeds → STOP.**

## Model 3: Claude-only

If both Codex and all Gemini models are unavailable:
- Inform user: "No external model available. Run `codex login` or `gemini auth login`."
- The main session can perform Claude-only analysis natively.

## Cleanup

Always run after every invocation, even on error:
```bash
rm -f /tmp/cross-model-input.txt /tmp/cross-model-output.md
```

## Return Format

Always include which model was actually used:
```markdown
- **External model**: [GPT-5.4 via Codex | Gemini 3.1 Pro | Gemini 2.5 Pro | Gemini 2.5 Flash | Gemini 3.1 Flash Lite | Claude-only]
- **Fallback used**: [no | yes — Codex unavailable, used Gemini 3.1 Pro | yes — Codex + Gemini 3.1 Pro failed, used Gemini 2.5 Flash]
```
