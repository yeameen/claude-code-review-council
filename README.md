# Multi-Agent Code Review for Claude Code

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin that delivers AI code review by running **six reviewers in parallel** on the same diff — then synthesizing all findings into a verified report.

Different model families miss different things. Run them all:

1. **Codex CLI** — GPT-5.5 at `xhigh` reasoning effort
2. **Gemini CLI** — Gemini 3.1 Pro
3. **Four Claude specialist subagents** — security, performance, logic, regression

The synthesis step de-duplicates findings, **verifies each one against the actual code** (reviewers hallucinate), tags severity, and shows you a unified report before applying any fixes.

## Install

```
/plugin marketplace add https://github.com/yeameen/claude-code-review-council
/plugin install review-council
```

## Usage

```
/review-council                    # review uncommitted changes
/review-council 1234               # review GitHub PR #1234
/review-council commit:abc123      # review a specific commit
```

The skill infers scope from context (uncommitted, branch vs main, specific commit, or PR number) — ask only when ambiguous.

## What you get

A synthesized report with:
- **Findings** tagged by source (which reviewer flagged it) and severity (P0–P3)
- **Verified citations** — every `file:line` opened and checked before being included
- **Disagreements** surfaced explicitly — where one reviewer said "fine" and another said "block merge"
- **False positives** dismissed with a one-line reason
- **Proposed actions** — wait for your approval before any code changes

## Why six reviewers?

Single-model reviews — even at flagship effort — have predictable blindspots:

- Holistic reviewers (Codex, Gemini) often miss **regression risk** because they don't grep aggressively for callers of changed functions.
- Specialist subagents catch axis-specific issues (e.g., race conditions, schema-breaking changes) that holistic reviewers sail past.
- Cross-family disagreement (GPT vs. Gemini vs. Claude) is signal — when all three agree, confidence is high; when they split, that's exactly where human judgment matters.

The four Claude specialists also serve as a fallback: if Codex or Gemini fail (quota errors, empty responses), the review still completes.

## Prerequisites

The skill calls out to external CLIs. Install and authenticate before use:

| Tool | Install |
|------|---------|
| [`codex`](https://github.com/openai/codex) | `npm i -g @openai/codex` |
| [`gemini`](https://github.com/google-gemini/gemini-cli) | `npm i -g @google/gemini-cli` |
| `gh` (for PR reviews by number) | [cli.github.com](https://cli.github.com/) |

If `codex` or `gemini` is missing or rate-limited, the skill degrades gracefully — the four Claude specialists carry the review on their own.

## Cost & time

3–8 minutes wall-time. Six flagship-effort reviewers cost real money. Use it for:

- Security-sensitive code
- Database migrations and schema changes
- Anything you'd want a senior engineer's eyes on before merging

Don't use it for typos, doc tweaks, or single-file refactors with passing tests. For those, use Claude Code's lighter `/pr-review` skill.

## Manual install (copy-paste)

If you don't want to use the marketplace flow:

```bash
git clone https://github.com/yeameen/claude-code-review-council
cp -r claude-code-review-council/skills/review-council ~/.claude/skills/
```

Then `/review-council` is available in any Claude Code session.

## License

MIT
