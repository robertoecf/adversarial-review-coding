---
name: adversarial-code-review
description: "Cross-model adversarial review of code, configs, and diffs. Spawns a background subagent that calls Codex CLI / Gemini for external red-team analysis, synthesizes findings, and returns unified critics and recommendations."
version: 0.4.0
model: inherit
allowed-tools: ["Read", "Grep", "Glob", "Bash", "Agent"]
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

Spawn a background subagent that independently reviews code via an
external model (Codex CLI / Gemini), cross-validates with its own analysis,
and returns ONLY unified critics and recommendations.

The main session is NOT blocked — it can continue working.

## How to Execute

### 1. Resolve the input

- **Inline code**: use directly
- **File path**: read the file first, include the content
- **"review uncommitted"**: note this for the agent (uses `codex review`)
- **"review --base main"**: note this for the agent

If no input: ask "What code should I review? Paste it, point to a file, or say 'review uncommitted'."
If input is a plan: suggest `/adversarial-plan-review`.
If input is a prompt: suggest `/prompt-optimize`.

### 2. Spawn the background subagent

Use the Agent tool with `run_in_background=true` and `model=sonnet`.

```
Agent(
    # inherits main session model (opus by default)
  run_in_background=true,
  prompt=<see below>
)
```

### Agent Prompt Template — Code Review

Build the agent prompt by replacing `{CODE_TEXT}` with the actual code:

---

You are an adversarial code reviewer (red-team). Your job:
1. Send this code to an external model for independent security/robustness analysis
2. Run your own red-team analysis
3. Cross-validate both analyses
4. Return ONLY the final unified critics and recommendations

## The Code to Review

{CODE_TEXT}

## Step 1: Call External Model

**CRITICAL: Call ONLY the FIRST available model. STOP as soon as one succeeds.
Do NOT call multiple models.**

### Write the template first:
```bash
cat << 'TEMPLATE_EOF' > /tmp/cross-model-input.txt
You are a red-team security and reliability analyst.
Assume everything will fail. Prove it with concrete exploit scenarios.

Review this code for:
1. SECURITY: injection, auth bypass, data exposure, OWASP top 10
2. ROBUSTNESS: race conditions, failure cascades, resource exhaustion, timeouts
3. COST: API scaling, token budgets, storage growth, compute complexity

---
{CODE_TEXT}
---

Per finding: P0-P3 severity, step-by-step attack/failure scenario,
likelihood, impact, exploit difficulty, mitigation A/B/C, recommendation.
Document exploit chains where findings combine.
TEMPLATE_EOF
```

### Try Codex CLI first:
```bash
which codex && test -f ~/.codex/auth.json && echo "CODEX" || echo "NO_CODEX"
```
If CODEX:
```bash
timeout 120 codex exec --full-auto --ephemeral -o /tmp/cross-model-output.md "$(cat /tmp/cross-model-input.txt)"
cat /tmp/cross-model-output.md
```
**If Codex succeeds → STOP. Go to Step 2. Do NOT call Gemini.**

### If Codex unavailable or fails, try Gemini cascade:
```bash
which gemini && test -f ~/.gemini/oauth_creds.json && echo "GEMINI" || echo "NO_GEMINI"
```
If GEMINI available, try models in order (stop at first success):
```bash
# 1. gemini-3.1-pro-preview
cat /tmp/cross-model-input.txt | timeout 120 gemini -p "" -y -m gemini-3.1-pro-preview > /tmp/cross-model-output.md 2>/dev/null
# If output is empty, try next:
# 2. gemini-3.1-flash-lite-preview
cat /tmp/cross-model-input.txt | timeout 120 gemini -p "" -y -m gemini-3.1-flash-lite-preview > /tmp/cross-model-output.md 2>/dev/null
# If empty, try next:
# 3. gemini-2.5-pro
cat /tmp/cross-model-input.txt | timeout 120 gemini -p "" -y -m gemini-2.5-pro > /tmp/cross-model-output.md 2>/dev/null
# If empty, try next:
# 4. gemini-2.5-flash
cat /tmp/cross-model-input.txt | timeout 120 gemini -p "" -y -m gemini-2.5-flash > /tmp/cross-model-output.md 2>/dev/null
```
**Stop at the first model that returns non-empty output.**

### If all unavailable: skip external model, do Claude-only analysis.

### Clean up:
```bash
rm -f /tmp/cross-model-input.txt /tmp/cross-model-output.md
```

### Agent Prompt Template — Git Review

If the user said "review uncommitted" or "review --base main":

---

You are an adversarial code reviewer. Your job:
1. Use Codex CLI's built-in review command
2. Run your own analysis on the same diff
3. Cross-validate and return unified findings

## Step 1: Get the diff and external review

```bash
codex review --uncommitted 2>&1 | tee /tmp/cross-model-output.md
cat /tmp/cross-model-output.md
rm -f /tmp/cross-model-output.md
```

Also read the diff yourself:
```bash
git diff
git diff --cached
```

---

## Step 2: Your Own Analysis

Independently review for:
- Security: injection, auth, data exposure, OWASP top 10
- Robustness: race conditions, failure cascades, resource exhaustion
- Cost: scaling, token budgets, storage growth
- Exploit chains: findings that combine into worse scenarios

## Step 3: Cross-Validate

- Agreements = `[cross-validated]` (high confidence)
- Only external = `[external-only]` (flag for review)
- Only you = `[claude-only]` (include with reasoning)
- Severity disagreements = take the higher

## Step 4: Return the Final Output

Return ONLY this — no raw dumps, no intermediate steps:

```markdown
## Adversarial Code Review
- **External model**: [GPT-5.4 via Codex | Gemini 2.5 Pro | Claude-only]
- **Review mode**: [template | codex review --uncommitted | codex review --base]
- **Findings**: [N total — X P0, Y P1, Z P2, W P3]

### Critics (severity-ordered)

#### [P0|P1|P2|P3]: Finding title [cross-validated | external-only | claude-only]
- **Attack/failure scenario**: step-by-step
- **Likelihood**: [low|medium|high]
- **Impact**: [low|medium|high|critical]
- **Recommendation**: specific fix with effort estimate

[repeat for each finding]

### Exploit Chains
[findings that combine into worse scenarios]

### Recommendations (priority-ordered)
1. [most critical fix]
2. [next fix]
```

---

### 3. Inform the user

After spawning the agent, tell the user:
"Adversarial code review running in background via [Codex/Gemini]. You'll be notified when it completes."
