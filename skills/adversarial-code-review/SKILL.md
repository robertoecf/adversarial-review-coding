---
name: adversarial-code-review
description: "Cross-model adversarial review of code, configs, and diffs. Sends code to an external model (Codex CLI / Gemini) for independent red-team analysis, then synthesizes findings with Claude's own analysis."
version: 0.3.1
model: inherit
allowed-tools: ["Read", "Grep", "Glob", "Bash"]
triggers:
  - "adversarial.?code"
  - "adversarial.?review"
  - "red.?team"
  - "security.?review"
  - "what.?could.?go.?wrong"
  - "find.?vulnerabilit"
  - "threat.?model"
  - "code.?review.?codex"
---

# Adversarial Code Review

Send the user's code to an external model for independent red-team
analysis, then synthesize the findings with your own analysis.

## Step 1: Resolve Input

- **Inline code**: use directly
- **File path**: read with Read tool
- **"review uncommitted"**: use `codex review --uncommitted` (skip template, go to Step 3b)
- **"review --base main"**: use `codex review --base main` (skip template, go to Step 3b)

If no input: ask "What code should I review? Paste it, point to a file, or say 'review uncommitted'."
If input is a plan: suggest `/adversarial-plan-review`.
If input is a prompt: suggest `/prompt-optimize`.

## Step 2: Check External Model Availability

```bash
which codex && test -f ~/.codex/auth.json && echo "CODEX" || echo "NO_CODEX"
```

If NO_CODEX:
```bash
which gemini && test -f ~/.gemini/oauth_creds.json && echo "GEMINI" || echo "NO_GEMINI"
```

If both unavailable: inform user, offer Claude-only code review.

## Step 3a: Template Flow (for inline/file code)

Write the filled template to a temp file:

```bash
cat << 'TEMPLATE_EOF' > /tmp/cross-model-input.txt
You are a red-team security and reliability analyst.
Assume everything will fail. Prove it with concrete exploit scenarios.

Review this code for:

1. SECURITY
   - Injection: SQL, command, template, prompt injection
   - Authentication/Authorization: missing auth, IDOR, privilege escalation
   - Data exposure: PII in logs, secrets in config, error message leakage
   - Input validation: missing sanitization, type confusion, boundary violations
   - OWASP Top 10 systematic check

2. ROBUSTNESS
   - Race conditions: TOCTOU, concurrent access, async hazards
   - Failure cascades: what breaks when dependency X is down?
   - Resource exhaustion: unbounded loops, memory leaks, connection pool drain
   - Timeout handling: missing timeouts, retry storms, no backoff

3. COST
   - API cost scaling: per-request costs at volume
   - Token budgets: unbounded context, no max_tokens
   - Storage growth: append-only, missing cleanup/TTL
   - Compute: O(n^2) or worse on growing data

---
<INSERT THE CODE HERE>
---

Per finding provide:
- Severity: P0 (critical) / P1 (high) / P2 (medium) / P3 (low)
- Attack/failure scenario: step-by-step path to exploitation or failure
- Likelihood: low/medium/high with evidence
- Impact: low/medium/high/critical with what breaks
- Exploit difficulty: trivial/moderate/hard
- Mitigation options A/B/C with effort and effectiveness
- Recommendation: specific choice with reasoning

If multiple findings combine into a worse scenario, document the exploit chain.
TEMPLATE_EOF
```

**If Codex:**
```bash
timeout 120 codex exec --full-auto --ephemeral -o /tmp/cross-model-output.md "$(cat /tmp/cross-model-input.txt)"
```

**If Gemini:**
```bash
cat /tmp/cross-model-input.txt | gemini -p "" -y -m gemini-2.5-pro > /tmp/cross-model-output.md 2>/dev/null
```

## Step 3b: Git Review Flow

```bash
codex review --uncommitted 2>&1 | tee /tmp/cross-model-output.md
```

Or for branch diffs:
```bash
codex review --base main 2>&1 | tee /tmp/cross-model-output.md
```

## Step 4: Capture Response

```bash
cat /tmp/cross-model-output.md
```

If empty/error and primary was Codex: try Gemini fallback.
If both fail: inform user, offer Claude-only review.

## Step 5: Clean Up

```bash
rm -f /tmp/cross-model-input.txt /tmp/cross-model-output.md
```

## Step 6: Synthesize

Present the external model's raw response, then cross-validate:

```markdown
## Cross-Model Code Review
- **External model**: [GPT-5.4 via Codex CLI | Gemini 2.5 Pro]
- **Review mode**: [template | codex review --uncommitted | codex review --base]
- **Fallback used**: [no | yes — reason]

### External Model Findings
[raw response, unmodified]

### Claude Cross-Validation

#### Cross-Validated (high confidence)
[findings both Claude and the external model agree on]

#### External-Only (needs review)
[findings only the external model caught — with Claude's assessment]

#### Claude-Only
[findings only Claude caught that the external model missed]

#### Exploit Chains
[findings that combine into worse scenarios]

#### Severity Disagreements
| Finding | External | Claude | Chosen |
|---------|----------|--------|--------|

### Recommendations (priority-ordered)
1. [most critical]
2. [next]
```

## Error Paths

| Condition | Response |
|-----------|----------|
| No input | Ask for code |
| Input is a plan | Suggest `/adversarial-plan-review` |
| Input is a prompt | Suggest `/prompt-optimize` |
| Codex + Gemini unavailable | Claude-only code review |
| Codex timeout | Try Gemini fallback |
| External model returns empty | Try fallback, then Claude-only |
| Code >2000 lines | Warn about cost, focus on highest-risk areas |
