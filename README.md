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

## How it works

The skill orchestrates six reviewers and a synthesis pass from a single Claude Code session:

**1. Build a workspace.** Saves the diff and a short `context.md` (scope, stated intent from PR description / commit messages, project conventions, out-of-scope items) to `/tmp/review-council-<timestamp>/`. All six reviewers see the same intent — not just the diff. This catches "implementation diverges from stated rule" findings that no-intent reviews miss.

**2. Launch six reviewers in parallel.** All started in the same turn, so wall-time is bounded by the slowest, not the sum:

| Reviewer | How it runs |
|---|---|
| **Codex CLI** (GPT-5.5 `xhigh`) | Backgrounded shell subprocess: `codex review --title "..." - <<<"$context+diff"` (prompt-mode — passes intent alongside the diff) |
| **Gemini CLI** (Gemini 3.1 Pro) | Backgrounded shell subprocess: `gemini -m gemini-3.1-pro-preview --yolo -p "$context+diff"` |
| **4 Claude specialists** | Spawned in parallel via Claude Code's `Agent` tool, one per axis (security / performance / logic / regression), each with a focused single-axis prompt and told to ignore findings outside its lane |

Each reviewer writes its report into the shared workspace dir.

**3. Synthesize.** Once all six reports land, the orchestrator:
- Deduplicates findings across all six streams
- **Verifies each citation by opening the file** — reviewers occasionally hallucinate `file:line` refs, so unverified ones get dropped
- Re-rates severity against what's actually in the code (a "P0" that's really a style nit gets downgraded; a "P3" that's a real race condition gets upgraded)
- Tags each finding with which reviewers flagged it — multiple independent flags = high confidence
- Surfaces disagreements explicitly when one reviewer said "fine" and another said "block"

**4. Push back when needed.** If a finding looks suspicious, the orchestrator can resume the relevant reviewer mid-flight rather than just dismissing it:
- Codex: `codex exec resume --last "you flagged X, but the code does Y — defend or retract"`
- Gemini: `gemini -r latest -p "<counter-evidence>"`
- Claude specialists: re-spawned with the disputed finding + counter-evidence

The full exchange is saved to the workspace for audit.

**5. Report and wait.** Presents a unified P0–P3 report with proposed actions. **No code changes until you approve.** Then applies fixes and re-runs your project's tests if cheap.

**Degradation:** If Codex or Gemini fails (quota errors, empty output, network issue), the skill notes it in the report and synthesizes from whatever did run. The four Claude specialists are reliable enough to carry a review on their own.

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
