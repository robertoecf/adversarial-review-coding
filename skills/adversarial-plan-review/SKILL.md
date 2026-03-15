---
name: adversarial-plan-review
description: "Cross-model adversarial review of implementation plans. Sends plans to an external model (Codex CLI / Gemini) for independent critique, then synthesizes findings with Claude's own analysis."
version: 0.3.1
model: inherit
allowed-tools: ["Read", "Grep", "Glob", "Bash"]
triggers:
  - "adversarial.?plan"
  - "review.?plan"
  - "validate.?plan"
  - "check.?plan"
  - "plan.?review"
  - "before.?implement"
---

# Adversarial Plan Review

Send the user's plan to an external model for independent adversarial
critique, then synthesize the findings with your own analysis.

## Step 1: Extract the Plan

Extract the raw plan text from the user's input.

If no plan provided: ask "What plan should I review? Paste it or point to a file."
If input is code: suggest `/adversarial-code-review` instead.

## Step 2: Check External Model Availability

```bash
which codex && test -f ~/.codex/auth.json && echo "CODEX" || echo "NO_CODEX"
```

If NO_CODEX:
```bash
which gemini && test -f ~/.gemini/oauth_creds.json && echo "GEMINI" || echo "NO_GEMINI"
```

If both unavailable: inform user, offer Claude-only plan review.

## Step 3: Fill Template and Call External Model

Write the filled template to a temp file:

```bash
cat << 'TEMPLATE_EOF' > /tmp/cross-model-input.txt
You are a senior engineering lead and adversarial reviewer.
Review this implementation plan. Assume it will fail. Prove it.

---
<INSERT THE PLAN TEXT HERE>
---

Validate for:
1. Scope alignment — does the plan match stated objectives? Scope creep? Under-scoped?
2. Missing steps — gaps in the implementation sequence? Testing? Migration?
3. Dependency ordering — can steps execute in stated order? Circular deps?
4. Rollback strategy — what happens if step N fails? Is each step reversible?
5. Blast radius — what existing functionality is at risk?
6. Success criteria — are there verifiable completion conditions?
7. Cost estimate — estimated complexity, files changed, test impact

Per finding provide:
- Severity: P0 (critical) / P1 (high) / P2 (medium) / P3 (low)
- Evidence: direct reference from the plan
- Problem: what's wrong and why it matters
- Recommendation: specific fix

Overall verdict: PROCEED / REVIEW_NEEDED / RETHINK
Provide an improved version of the plan incorporating all fixes.
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

## Step 4: Capture Response

```bash
cat /tmp/cross-model-output.md
```

If empty or error and primary was Codex: try Gemini fallback.
If both fail: inform user, offer Claude-only review.

## Step 5: Clean Up

```bash
rm -f /tmp/cross-model-input.txt /tmp/cross-model-output.md
```

## Step 6: Synthesize

Present the external model's raw response, then cross-validate:

```markdown
## Cross-Model Plan Review
- **External model**: [GPT-5.4 via Codex CLI | Gemini 2.5 Pro]
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

#### Severity Disagreements
| Finding | External | Claude | Chosen |
|---------|----------|--------|--------|

### Unified Verdict: [PROCEED | REVIEW_NEEDED | RETHINK]

### Recommendations (priority-ordered)
1. [most critical]
2. [next]
```

## Error Paths

| Condition | Response |
|-----------|----------|
| No input | Ask for the plan |
| Input is code | Suggest `/adversarial-code-review` |
| Codex + Gemini unavailable | Claude-only plan review |
| Codex timeout | Try Gemini fallback |
| External model returns empty | Try fallback, then Claude-only |
