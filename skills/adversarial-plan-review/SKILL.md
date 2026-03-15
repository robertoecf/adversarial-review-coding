---
name: adversarial-plan-review
description: "Cross-model adversarial review of implementation plans. Spawns a background subagent that calls Codex CLI / Gemini for external critique, synthesizes findings, and returns a revised plan with critics and recommendations."
version: 0.4.0
model: inherit
allowed-tools: ["Read", "Grep", "Glob", "Bash", "Agent"]
triggers:
  - "adversarial.?plan"
  - "review.?plan"
  - "validate.?plan"
  - "check.?plan"
  - "plan.?review"
  - "before.?implement"
---

# Adversarial Plan Review

Spawn a background subagent that independently reviews the plan via an
external model (Codex CLI / Gemini), cross-validates with its own analysis,
and returns ONLY the final revised plan with critics and recommendations.

The main session is NOT blocked — it can continue working.

## How to Execute

### 1. Extract the plan text from the user's input

If no plan: ask "What plan should I review? Paste it or point to a file."
If input is code: suggest `/adversarial-code-review` instead.

### 2. Spawn the background subagent

Use the Agent tool with `run_in_background=true` and `model=sonnet`.

The Agent prompt MUST contain:
- The complete skill instructions (copied below)
- The full plan text
- Nothing else — no conversation history, no context from the main session

```
Agent(
    # inherits main session model (opus by default)
  run_in_background=true,
  prompt=<see below>
)
```

### Agent Prompt Template

Build the agent prompt by replacing `{PLAN_TEXT}` with the user's actual plan:

---

You are an adversarial plan reviewer. Your job:
1. Send this plan to an external model for independent critique
2. Run your own plan validation
3. Cross-validate both analyses
4. Return ONLY the final revised plan with critics and recommendations

## The Plan to Review

{PLAN_TEXT}

## Step 1: Call External Model

Check availability and call the first available model:

Primary — Codex CLI:
```bash
which codex && test -f ~/.codex/auth.json && echo "CODEX" || echo "NO_CODEX"
```

If CODEX available:
```bash
cat << 'TEMPLATE_EOF' > /tmp/cross-model-input.txt
You are a senior engineering lead and adversarial reviewer.
Review this implementation plan. Assume it will fail. Prove it.

---
{PLAN_TEXT}
---

Validate for:
1. Scope alignment — does the plan match stated objectives? Scope creep? Under-scoped?
2. Missing steps — gaps in the implementation sequence? Testing? Migration?
3. Dependency ordering — can steps execute in stated order? Circular deps?
4. Rollback strategy — what happens if step N fails? Is each step reversible?
5. Blast radius — what existing functionality is at risk?
6. Success criteria — are there verifiable completion conditions?
7. Cost estimate — estimated complexity, files changed, test impact

Per finding: P0-P3 severity, evidence, problem, recommendation.
Verdict: PROCEED / REVIEW_NEEDED / RETHINK
Provide an improved version of the plan incorporating all fixes.
TEMPLATE_EOF
timeout 120 codex exec --full-auto --ephemeral -o /tmp/cross-model-output.md "$(cat /tmp/cross-model-input.txt)"
cat /tmp/cross-model-output.md
rm -f /tmp/cross-model-input.txt /tmp/cross-model-output.md
```

Fallback — Gemini CLI:
```bash
which gemini && test -f ~/.gemini/oauth_creds.json && echo "GEMINI" || echo "NO_GEMINI"
```
If GEMINI available, pipe the same template to:
```bash
cat /tmp/cross-model-input.txt | gemini -p "" -y -m gemini-2.5-pro
```

If both unavailable: skip external model, do Claude-only analysis.

## Step 2: Your Own Analysis

Independently validate the plan for:
- Scope alignment vs objectives
- Missing steps (testing, migration, rollback)
- Dependency ordering
- Rollback strategy per step
- Blast radius on existing functionality
- Success criteria definition

## Step 3: Cross-Validate

Compare your findings with the external model's findings:
- Agreements = high confidence
- Only external model caught = flag for review
- Only you caught = include with reasoning
- Severity disagreements = take the higher

## Step 4: Return the Final Output

Return ONLY this — no raw dumps, no intermediate steps:

```markdown
## Adversarial Plan Review
- **External model**: [GPT-5.4 via Codex | Gemini 2.5 Pro | Claude-only]
- **Verdict**: [PROCEED | REVIEW_NEEDED | RETHINK]
- **Findings**: [N total — X P0, Y P1, Z P2, W P3]

### Critics

#### [P0|P1|P2|P3]: Finding title [cross-validated | external-only | claude-only]
- **Problem**: what's wrong and why it matters
- **Evidence**: reference from the plan
- **Recommendation**: specific fix

[repeat for each finding, highest severity first]

### Revised Plan

[the complete improved plan incorporating all recommendations,
ready to execute — not a diff, the full revised plan]

### Key Changes from Original
1. [change] — [why]
2. [change] — [why]
```

---

### 3. Inform the user

After spawning the agent, tell the user:
"Adversarial plan review running in background via [Codex/Gemini]. You'll be notified when it completes."
