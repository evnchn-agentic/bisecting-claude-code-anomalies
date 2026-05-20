---
name: bisecting-claude-code-anomalies
description: Use when a user reports Claude Code "feels different", "drunk", "less sharp", "off", "got worse", "got dumber", "degradation", "quality drop", "kinda weird the last couple of days", "harder to follow my requests", or otherwise suspects vibe drift / behavioral change across CC versions despite using the same model and saying "no model changes on my end". Also use when investigating whether a specific CC behavior shift is harness-side (system prompt, tools, CHANGELOG) versus model-side (Anthropic training nudges, sycophancy tweaks, quantization rumors). Triggers: "behavioural differences with time", "context dilution", "started doing X yesterday", "weirdly verbose / terse", "started refusing X", "agreement language gone", "stop hook regression", instruction-following regression complaints, HN/Reddit threads asking "is Claude getting worse".
---

# Bisecting Claude Code anomalies

## Overview

When a user says Claude Code is behaving differently despite "same model", the question is: did the **harness** (system prompt, tools, CHANGELOG) change, or did the **model** change? The harness side is detectable by diffing community-extracted system prompts against your installed version. The model side is not — but identifying *which* side a behavior lives on is the key value.

**Core insight:** local JSONL `version` field is ground truth for which Claude Code version was active at a specific wall-clock moment. The Piebald-AI repo is ground truth for what the system prompt looked like at that version. Together they answer "what changed under me right before I noticed something off."

This skill is **pure inline** — each of the 7 steps below is one or two shell commands. Run only the ones you need; skip steps where the signal source isn't reachable (corporate egress, missing tool, etc.). No bundled script, no required filesystem layout, no clobberable baselines.

## When NOT to use

- Project-specific bugs — use `superpowers:systematic-debugging` instead.
- One-shot prompt issues, fixable by rewording — no harness drift to bisect.
- Model-name swaps the user did intentionally (`--model` changed, /model switched).
- Suspected MCP / hook / settings.json regression — check those config surfaces directly first; bisect only if config is unchanged.

## Inputs (all ethical — no source leaks)

- **`Piebald-AI/claude-code-system-prompts`** — community-reversed system prompts, near-daily updates. Commit titles include `(+N tokens)` deltas — gold for spotting bloat. Cloned by `cc-drift.sh` on first run.
- **`anthropics/claude-code` CHANGELOG.md** — official per-version notes.
- **`claude --version`** — what's actually installed.
- **`~/.claude/projects/**/*.jsonl`** — every CC message carries a `.version` field. This is the per-session, per-message authoritative version log.

**Forbidden:** leaked source-code repositories. Ethically out, and stale relative to Piebald.

## The 7-step bisect

1. **Pin the user's "smell" to a wall-clock moment.** "Feels off lately" is unactionable. Force a timestamp: screenshot pixel-clock, Outlook/Slack message ts, conversation export. Convert local time to UTC.

2. **Look up which CC version was active at that UTC moment** by scanning local JSONLs. Example for an incident around 2026-05-08 16:41 UTC:
   ```bash
   jq -r 'select(.timestamp >= "2026-05-08T16:30" and .timestamp < "2026-05-08T17:00")
          | .version' ~/.claude/projects/*/*.jsonl 2>/dev/null | sort -u
   ```
   The user may have been running multiple versions concurrently across sessions — note all of them.

3. **Refresh the Piebald repo and bracket the suspect window.** A change attributed to version N may have been "baked in" at N-1. Always bisect the bracket `[N-1, N]` at minimum, `[N-2, N]` when safe. First time on this machine — pick any path you have write access to:
   ```bash
   git clone --depth 100 https://github.com/Piebald-AI/claude-code-system-prompts.git /path/to/piebald
   ```
   Subsequent runs:
   ```bash
   git -C /path/to/piebald fetch --quiet && git -C /path/to/piebald reset --hard origin/main
   ```

4. **Rank candidate commits by token delta.** Piebald commit titles include `(+N tokens)`. The biggest delta in the window is almost always the prior:
   ```bash
   git -C ~/cc-drift/state/piebald log --oneline --grep='tokens' v<N-2>..v<N>
   ```
   Order-of-magnitude differences are the smoking gun — a single `+6720` next to a row of `±200`s answers the question.

5. **Read the actual diff** for the leading candidate. Look for new product-feature documentation (Managed Agents, MCP, tool-use sections) that bloats the prompt without serving this user's workflow. Quote 2-3 concrete lines from the diff in your answer.

6. **Cross-reference `anthropic/claude-code` CHANGELOG.md.** Often mentions the harness-side change without naming the system-prompt one (e.g. "improved stop hooks" can correspond to evaluator-prompt addition). Treat the CHANGELOG as corroboration, not the primary signal.

7. **State plainly what the diff CANNOT explain.** Sycophancy, agreement language ("Yes - strong agree"), refusal style, tone shifts are **model-side** — invisible to system-prompt diffing. Flag the unknown. Do not over-claim that you bisected something you can't see.

## Worked example: 2.1.132 Managed Agents bloat (2026-05-09)

- **Complaint:** "Claude Code felt drunk, not the sharpest knife." Pinned to 2026-05-09 00:41 HKT = 2026-05-08 16:41 UTC.
- **Version active:** JSONL scan showed concurrent 2.1.131 / 2.1.133 sessions at the moment. 2.1.132 lives between.
- **Token deltas in window:** every entry ≤ ±525 tokens **except 2.1.131 → 2.1.132 = +6,720 tokens**. Smoking gun by an order of magnitude.
- **Diff content:** ~440 new lines of `data-managed-agents-*` documentation — multiagent sessions, outcomes, webhooks, endpoint reference. User does not use Managed Agents.
- **Mechanism:** context dilution on xhigh thinking effort.
- **What the diff could NOT explain:** the disappearance of "Yes - strong agree" / "You're absolutely right" stylistic markers between 131 and 136. Plausibly Anthropic-side model nudge, or downstream effect of bloat starving stylistic budget. Flagged as unknown — not claimed as bisected.

## Quick reference

| Signal | Source | Latency | Tells you |
|---|---|---|---|
| Local CC version | `claude --version` | live | What you actually run |
| JSONL version histogram | `~/.claude/projects/**/*.jsonl` `.version` field | live | Which versions touched which sessions, when |
| System-prompt diff | Piebald-AI repo | ~same-day | Harness-side prompt changes |
| Official notes | `anthropic/claude-code` CHANGELOG | per release | What Anthropic disclosed |
| Model-side training | NONE | N/A | Inferred only by elimination |

## Common mistakes

- **Treating "feels off" as the input.** Force a timestamp first. No anchor → no bisect.
- **Looking at leaked source-code repos.** Ethically out, and stale; Piebald is the right surface.
- **Reading the CURRENT installed version's prompt.** The drift may have happened on a version the user already moved off of. Always check JSONLs at the incident's timestamp, not at investigation time.
- **Skipping the ±1-version bracket.** Piebald may have committed the change between the prior and the version active at incident — it's "baked in" to the active version but introduced in the previous bump.
- **Attributing all behavior to the system prompt.** Sycophancy, refusal patterns, tone, and agreement language are model-side and bisect-invisible. Acknowledge the gap.
- **Running the tool on every turn.** It clones / fetches / curls. Run it once per investigation, cache the report, reference the report by path.

## Reporting back

Deliverable is a one-screen answer that turns "smelling something wrong" into a specific change citation:

- Version transition (e.g. `2.1.131 → 2.1.132`)
- Token delta (e.g. `+6,720`, with neighbors for scale)
- 3-bullet diff summary
- Plain statement of which observed behaviors the diff can vs cannot explain
- If model-side: "this is invisible to bisecting; can only be confirmed by Anthropic disclosure or cross-user pattern."

## Related public tools (verified 2026-05-20)

- **`Piebald-AI/claude-code-system-prompts`** — primary signal source for this skill. Solo-maintained, ~same-day cadence per CC release, 181 versions tracked at audit. Only continuously-maintained extractor in public.
- **`Piebald-AI/tweakcc`** (~2k stars) — same maintainer; does HTML side-by-side diffs of CC system prompts across versions. Aimed at conflict-resolution for users who patch the prompt, not at behavior attribution. Useful adjunct if you want a polished diff viewer instead of `git diff`.
- **`marginlab.ai`** — external daily-benchmark dashboard tracking output-side regressions across LLM versions. Complement, not competitor: measures *outputs*, this skill inspects *prompts*. Cite when the user asks "is anyone else seeing this?"
- **`zep-us/claude-system-prompt`** — dormant since Feb 2026, last covered v2.1.34. Don't reach for; ~3 months stale by mid-2026.
- **Anthropic's April 23 2026 postmortem** — only official disclosure of a specific CC prompt change (the "25 words / 100 words" verbosity rule). No version-by-version prompt publishing from Anthropic. Cite when the user expects Anthropic to be the source of truth.

Community vernacular for the symptom this skill addresses: "degradation", "got worse", "quality drop", "got dumber". Evan's "drunk Claude" is his idiom; recognize both.

## Provenance

Methodology distilled 2026-05-19 from the 2.1.132 Managed Agents bloat bisect (see worked example above). Skill is shell-only — no bundled scripts, no required state directory, no telemetry. Compose with `superpowers:systematic-debugging` when the anomaly turns out to be project-local rather than CC-version-wide.
