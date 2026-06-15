---
name: autoreview
description: "Run a structured code review (Cursor agent default with gpt-5.5-high, optional Claude, Codex, Pi, Droid, Copilot, or OpenCode) as a closeout check on a local or PR branch before commit or ship. Use this whenever the user asks for a code review, agent review, cursor review, gpt-5.5 review, second-model review, or autoreview, and proactively after non-trivial code edits before finalizing, committing, pushing, or opening a PR."
---

# Auto Review

Run the bundled structured review helper as a closeout check. This is code review, not approval routing.

The `agent` engine (Cursor's `cursor-agent` CLI) is the default when no engine is set. It uses `gpt-5.5-high` by default, runs read-only, and should remain the normal final closeout engine. Claude review is the strongly-isolated alternative and uses `claude-fable-5` by default.

Use when:

- the user asks for an agent review / cursor review / gpt-5.5 review / Claude review / Codex review / Pi review / Droid review / OpenCode review / autoreview / second-model review
- after non-trivial code edits, before final/commit/ship
- reviewing a local branch or PR branch after fixes

## Contract

- Treat review output as advisory. Never blindly apply it.
- Verify every finding by reading the real code path and adjacent files.
- Read dependency docs/source/types when the finding depends on external behavior.
- Reject unrealistic edge cases, speculative risks, broad rewrites, and fixes that over-complicate the codebase.
- Prefer small fixes at the right ownership boundary; no refactor unless it clearly improves the bug class.
- When an accepted finding shows a bug class or repeated pattern, inspect the current PR scope for sibling instances before fixing.
- Fix the scoped bug class at once when practical; stop at touched surfaces, owner boundaries, and clear follow-up territory.
- Keep going until structured review returns no accepted/actionable findings.
- If a review-triggered fix changes code, rerun focused tests and rerun the structured review helper.
- Never switch or override the requested review engine/model. If the review hits model capacity, retry the same command a few times with the same engine/model.
- Be patient with large bundles. Structured review can take many minutes while the model call is active, especially with read-only tool use or web search enabled.
- Treat heartbeat lines like `review still running: agent elapsed=... pid=...` as healthy progress, not a hang. Pass `--stream-engine-output` when live engine text is useful; the agent and Claude displays filter thinking/tool chatter, other engines pass raw output through.
- Do not kill a review just because it has been quiet for a few minutes. Inspect the process only after missing multiple expected heartbeats or after an obviously failed subprocess; prefer letting the same helper command finish.
- Read-only tools are useful in review mode. The helper runs `cursor-agent` in `--mode ask` (read-only) so the reviewer can inspect files and dependency contracts without editing anything.
- Security perspective is always included, but it should not cripple legitimate functionality. Report security findings only when the change creates a concrete, actionable risk or removes an important safety check.
- Do not invoke nested reviewers or reviewer panels from inside the review. The helper builds one bundle, calls one selected engine, validates one structured result, and stops.
- Stop as soon as the helper exits 0 with no accepted/actionable findings. Do not run an extra review just to get a nicer "clean" line or a second opinion.
- Treat the helper's successful exit plus absence of actionable findings as the clean review result, even if the underlying engine output is terse.
- Multi-reviewer panels are opt-in only. Use them when explicitly requested or when risk justifies the extra spend; the main agent still verifies every accepted finding before fixing.
- If rejecting a finding as intentional/not worth fixing, add a brief inline code comment only when it explains a real invariant or ownership decision that future reviewers should know.
- Do not push just to review. Push only when the user requested push/ship/PR update.

## Engine isolation reality

When autoreview runs inside the repository under review, the reviewer should not let the reviewed branch alter the review.

- The **agent** engine (`cursor-agent`) runs read-only via `--mode ask` with `--trust --workspace <repo>`, so it cannot edit files or run mutating shell, and it never receives `--force`/`--yolo`. **But** `cursor-agent` still loads your user and project Cursor configuration — `.cursor/rules`, `AGENTS.md`, installed skills and plugins — and exposes no flag to disable reviewed-repo rule/skill loading. So the agent engine has weaker prompt-injection isolation than codex/claude: read-only mode bounds the blast radius, but a hostile branch could still influence the reviewer's instructions. Prefer the **claude** engine for untrusted code, or review such branches in a clean checkout.
- The **claude** engine runs with `--safe-mode --setting-sources user --strict-mcp-config --disallowedTools mcp__*`, which disables reviewed-repo hooks/skills/plugins/MCP/CLAUDE.md while keeping normal auth. This is the strongest isolation available here.
- **codex**, **pi**, and **opencode** carry their own isolation flags (see the original engine matrix); they are available but not installed by default in this environment.

## Skill Path (set once)

Set the skill script paths once, then use `"$AUTOREVIEW"` and `"$AUTOREVIEW_HARNESS"` in the examples below.

```bash
# Global skill (Claude Code):
export AUTOREVIEW="$HOME/.claude/skills/autoreview/scripts/autoreview"
export AUTOREVIEW_HARNESS="$HOME/.claude/skills/autoreview/scripts/test-review-harness"
```

```bash
# Project-local skill in the current repo:
export AUTOREVIEW=".claude/skills/autoreview/scripts/autoreview"
export AUTOREVIEW_HARNESS=".claude/skills/autoreview/scripts/test-review-harness"
```

The helper is a Python 3 script; run it directly (it is executable) or as `python3 "$AUTOREVIEW"`.

## Pick Target

Dirty local work:

```bash
"$AUTOREVIEW" --mode local
```

Use this only when the patch is actually unstaged/staged/untracked in the current checkout. `--mode uncommitted` is accepted as an alias for `--mode local`. For committed, pushed, or PR work, point the helper at the commit or branch diff instead; do not force dirty modes. A clean local review only proves there is no local patch.

Branch/PR work:

```bash
"$AUTOREVIEW" --mode branch --base origin/main
```

Optional review context is first-class:

```bash
"$AUTOREVIEW" --mode branch --base origin/main --prompt-file /tmp/review-notes.md --dataset /tmp/evidence.json
```

If an open PR exists, use its actual base:

```bash
base=$(gh pr view --json baseRefName --jq .baseRefName)
"$AUTOREVIEW" --mode branch --base "origin/$base"
```

Committed single change:

```bash
"$AUTOREVIEW" --mode commit --commit HEAD
```

Use commit review for already-landed or already-pushed work on `main`. Reviewing clean `main` against `origin/main` is usually an empty diff after push. For a small stack, review each commit explicitly or review the branch before merging with `--base`.

## Parallel Closeout

Format first if formatting can change line locations. Then it is OK to run tests and review in parallel:

```bash
"$AUTOREVIEW" --parallel-tests "<focused test command>"
```

Tradeoff: tests may force code changes that stale the review. If tests or review lead to code edits, rerun the affected tests and rerun review until no accepted/actionable findings remain. Once that rerun exits cleanly, stop.

## Review Panels

Run multiple reviewers against one frozen bundle:

```bash
"$AUTOREVIEW" --reviewers agent,claude
```

`--panel` is shorthand for the agent plus Claude unless `--engine` changes the first reviewer:

```bash
"$AUTOREVIEW" --panel
```

Set reviewer models explicitly. The agent engine bakes reasoning effort into the model id, so pick the effort by choosing the model:

```bash
"$AUTOREVIEW" --reviewers agent,claude --model agent=gpt-5.5-high --model claude=claude-fable-5 --thinking claude=max
```

Inline syntax is also supported for simple model IDs:

```bash
"$AUTOREVIEW" --reviewers agent:gpt-5.5-high,claude:claude-fable-5:max
```

## Models and effort

The helper accepts `--model` globally or per engine (`engine=model`). The **agent** engine has no `--thinking` knob — reasoning effort is part of the Cursor model id. Use `--model` to choose it.

Recommended model defaults:

| Engine | Default model | Note |
|--------|---------------|------|
| **agent** (default) | `gpt-5.5-high` | Cursor's `cursor-agent`; "GPT-5.5 1M High" |
| **claude** | `claude-fable-5` | strongest reviewed-repo isolation here |

CLI flags and environment variables override these defaults.

GPT-5.5 effort variants available to `cursor-agent` (pick via `--model`):

| Model id | Cursor label |
|----------|--------------|
| `gpt-5.5-none` | GPT-5.5 1M None |
| `gpt-5.5-low` | GPT-5.5 1M Low |
| `gpt-5.5-medium` | GPT-5.5 1M |
| `gpt-5.5-high` (default) | GPT-5.5 1M High |
| `gpt-5.5-extra-high` | GPT-5.5 1M Extra High |

Run `cursor-agent --list-models` to see the full catalog (Claude, Gemini, Composer, and Codex models are also available there). The `--thinking` flag still applies to the non-agent engines (Claude: `low/medium/high/xhigh/max`; Codex: `none/minimal/low/medium/high/xhigh`; etc.).

### Environment defaults

CLI flags take precedence over environment variables.

| Variable | Purpose |
|----------|---------|
| `AUTOREVIEW_ENGINE` | Default engine (defaults to `agent`) |
| `CURSOR_AGENT_BIN` | Path to the `cursor-agent` binary (defaults to `cursor-agent` on PATH) |
| `AUTOREVIEW_MODEL` | Override the built-in default `--model` for all engines |
| `AUTOREVIEW_AGENT_MODEL` | Per-engine model override for the agent engine, e.g. `gpt-5.5-extra-high` |
| `AUTOREVIEW_THINKING` | Default `--thinking` for engines that support it (ignored by agent) |
| `AUTOREVIEW_STREAM_ENGINE_OUTPUT` | `1` to stream live engine text |

## How the agent engine runs

The helper invokes `cursor-agent` like this (prompt delivered over stdin, never argv):

```bash
cursor-agent --print \
  --output-format json \         # stream-json when --stream-engine-output is set
  --mode ask \                   # read-only Q&A: inspect, do not edit
  --trust \                      # headless trust grant so it does not block on a prompt
  --workspace <repo> \
  --model gpt-5.5-high
```

The model is instructed to return exactly one JSON object matching the review schema. The helper extracts that object from the `result` field of `cursor-agent`'s JSON (or the final `result` event in stream-json), validates it, drops out-of-scope findings, prints the report, and exits nonzero when there are accepted/actionable findings.

## Helper

```bash
"$AUTOREVIEW" --help
```

The helper:

- chooses dirty local changes first; accepts `--mode uncommitted` as an alias for `--mode local`
- otherwise uses the current PR base if `gh pr view` works, else `origin/main` for non-main branches
- supports `--engine agent`, `claude`, `codex`, `droid`, `copilot`, `pi`, and `opencode`; default is `AUTOREVIEW_ENGINE` or `agent`
- use `--mode commit --commit <ref>` for already-committed work, especially clean `main` after landing
- writes only to stdout unless `--output`, `--json-output`, or live streamed engine stderr is set
- supports `--dry-run`, `--parallel-tests`, `--parallel-tests-shell`, `--prompt`, `--prompt-file`, `--dataset`, `--no-web-search`, and commit refs
- supports `--stream-engine-output` (or `AUTOREVIEW_STREAM_ENGINE_OUTPUT=1`) for live engine text while preserving structured validation; the agent display shows assistant text and final token usage and collapses thinking/tool events
- supports opt-in review panels with `--panel` / `--reviewers`, plus per-engine `--model`
- runs the agent engine read-only (`--mode ask`, no `--force`); `--no-tools` is rejected for agent — use `--engine claude --no-tools` for a no-tools run
- prints `review still running: <engine> elapsed=<seconds>s pid=<pid>` to stderr while waiting
- prints `autoreview clean: no accepted/actionable findings reported` when the selected review command exits 0
- exits nonzero when accepted/actionable findings are present

### Smoke harness

The smoke harness builds a temporary repo with a deliberately unsafe patch or a security-sensitive-but-safe patch, then verifies the selected engine through autoreview:

```bash
"$AUTOREVIEW_HARNESS" --fixture malicious --engine agent
"$AUTOREVIEW_HARNESS" --fixture benign --engine agent
```

The malicious fixture requires the review to flag the injected command-execution bug; the benign fixture checks the reviewer does not flag safe filesystem/exec/auth-adjacent code.

## Final Report

Include:

- review command used
- tests/proof run
- findings accepted/rejected, briefly why
- the clean review result from the final helper/review run, or why a remaining finding was consciously rejected

Do not run another review solely to improve the final report wording. If the final helper run exited 0 and produced no accepted/actionable findings, report that exact run as clean.
