---
name: bisecting-claude-code-anomalies
description: Use when a user reports Claude Code "feels different", "drunk", "less sharp", "off", "got worse", "got dumber", "degradation", "quality drop", "kinda weird the last couple of days", "harder to follow my requests", "refuses too often now", "sycophancy changed", "reward hacking", "agreement language gone", or otherwise suspects behavioral change across CC versions. Also use when investigating whether a specific behavior shift is harness-side (system prompt, tools, CHANGELOG) versus model-side (Anthropic training nudges, sycophancy, reward-hacking, refusal style). Triggers: "behavioural differences with time", "context dilution", "started doing X yesterday", "weirdly verbose / terse", "started refusing X", "stop hook regression", "gamed the test instead of fixing code", "lied about what it ran", "unsolicited high-impact action", instruction-following regression complaints, HN/Reddit threads asking "is Claude getting worse".
---

# Bisecting Claude anomalies

## Overview

When Claude Code behaves differently, there are **three** possible culprits — not two: your own **config** (CLAUDE.md / skills / hooks / MCP), the **harness** (system prompt, tools, CHANGELOG), or the **model**. Rule them out in that order — cheapest and most-in-your-control first.

**Phase 0** — config side: reproduce under `--safe-mode` (all customizations off). One command; decides in-one-shot whether the drift is yours or the base install's. Run this FIRST — the harness/model bisects are wasted effort if a bloated CLAUDE.md or a bad hook is the real cause.

**Phase A** — harness side: diff the system prompt via Piebald-AI. Fast, concrete, shell-only. Its signal lives in JSONL `.version`.

**Phase B** — model side: diff the behavioral shadow via Anthropic system cards. Kicks in when Phase A finds nothing or can't explain the full behavior. Its signal lives in JSONL `.message.model`. O(k) — shenanigan-first, never a generic two-card diff.

This skill is **pure inline** — each step is one or two shell commands. Run only the ones you need.

## When NOT to use

- Project-specific bugs — use `superpowers:systematic-debugging` instead.
- One-shot prompt issues fixable by rewording — no drift to bisect.
- Model-name swaps the user did intentionally (`--model` changed, /model switched).
- Suspected MCP / hook / settings.json regression — Phase 0 (`--safe-mode`) isolates these in one command; run it, then check the offending customization directly.

## Inputs

**Phase A (harness)**
- **`Piebald-AI/claude-code-system-prompts`** — community-reversed system prompts, near-daily updates. Commit titles include `(+N tokens)` deltas.
- **`anthropics/claude-code` CHANGELOG.md** — official per-version notes.
- **`claude --version`** — what's actually installed.
- **JSONL `.version` field** — per-message authoritative harness version log.

**Phase B (model)**
- **JSONL `.message.model` field** — per-message authoritative model string. The model-side analogue of `.version`.
- **Anthropic system cards** — `anthropic.com/<model>-system-card` and the `assets.anthropic.com` PDF mirror. Full cards for major bumps; *some* minor bumps (e.g. Opus 4.1) ship an **addendum** with comparative tables — when present it's your O(1) gift, but check metric direction, error bars, and any methodology note (some deltas are within noise).
- **`marginlab.ai`** — cross-user output regression dashboard. Answers "is anyone else seeing this?"

**Forbidden:** leaked source-code repos (ethically out, stale). Speculation dressed as a card finding. Non-deployed preview models — your sessions ran deployed models; a card for a model you can't call explains nothing.

## Phase 0 — Clean-room isolation with `--safe-mode` (1 command)

Before touching the JSONL, rule out the culprit it **can't** see: your own customizations. A bloated `CLAUDE.md`, a heavy skill, a misfiring hook, or MCP tool-count bloat (a 200k window shrunk to ~70k of usable context) all read as "drunk Claude" — and none of them are the base harness or the model.

`--safe-mode` (CC ≥ 2.1.x) starts with **all** customizations disabled — CLAUDE.md, skills, plugins, hooks, MCP servers, custom commands/agents, output styles, workflows, themes, keybindings — while **auth, model selection, built-in tools, and permissions work normally** (it sets `CLAUDE_CODE_SAFE_MODE=1`; admin-managed policy settings still apply). That makes it a clean room: same model, same harness, your customization surfaces disabled (permissions and admin policy still apply).

```bash
claude --safe-mode      # then reproduce the anomaly here
```

- **Anomaly VANISHES in safe mode** → the anomaly is **customization-dependent**. Bisect your own config FIRST: disable half your skills/hooks/MCP servers (or trim CLAUDE.md), re-test, repeat until you've minimized the offender. This is the `memory-gc` / context-bloat lineage of the drift. Usually you're done here — but note safe mode proves *dependence on your config*, not that config is the *sole* root cause: a base harness/model regression can surface only in combination with a file/hook you load. So if the minimized offender looks innocent on its own (a tiny CLAUDE.md line, a benign skill), suspect an interaction and still run Phase A/B.
- **Anomaly SURVIVES in safe mode** → your config is exonerated as a *standalone* cause; the drift is in the base install (or a base × nothing-of-yours path) → **proceed to Phase A.**

Bonus: safe mode is also the **clean room for Phase B reproduction (step 9)** — reproducing under `--safe-mode` strips your `CLAUDE.md`/skills so they can't confound the card lookup. A `CLAUDE.md`-as-constitution refusal (routing table, last row) is exactly this confound: if it normalizes in safe mode, the *file* was the cause, not the model.

## Phase A — Harness bisect (7 steps)

1. **Pin the "smell" to a wall-clock moment.** "Feels off lately" is unactionable. Force a timestamp: screenshot pixel-clock, Slack/Outlook ts, conversation export. Convert local time to UTC.

2. **Look up which CC version was active at that moment** via JSONL:
   ```bash
   jq -r 'select(.timestamp >= "2026-05-08T16:30" and .timestamp < "2026-05-08T17:00")
          | .version' ~/.claude/projects/*/*.jsonl 2>/dev/null | sort -u
   ```
   Note all versions — the user may have straddled a rollover across concurrent sessions.

3. **Refresh Piebald and bracket the suspect window.** A change in version N is often "baked in" at N-1 — bracket `[N-1, N]` at minimum, `[N-2, N]` when unsure:
   ```bash
   git clone --depth 100 https://github.com/Piebald-AI/claude-code-system-prompts.git /path/to/piebald
   # subsequent runs:
   git -C /path/to/piebald fetch --quiet && git -C /path/to/piebald reset --hard origin/main
   ```

4. **Rank candidate commits by token delta.** The biggest delta in the window is almost always the culprit:
   ```bash
   git -C /path/to/piebald log --oneline --grep='tokens' v<N-2>..v<N>
   ```
   An order-of-magnitude outlier (`+6,720` next to a row of `±200`s) is a smoking gun.

5. **Read the actual diff** for the leading candidate. Quote 2-3 concrete lines. Look for new product-feature documentation that bloats the prompt without serving this user's workflow.

6. **Cross-reference the official CHANGELOG.** Corroboration, not the primary signal — it often names the intent without naming the prompt change.

7. **State what the diff explains and what it does not.** If the behavior is fully explained: done. If the diff is null or leaves behavior unexplained (sycophancy, refusal style, agreement language, reward-hacking) — **proceed to Phase B**.

## Phase B — Model bisect (4 steps)

Phase B is O(k) where k = number of shenanigans observed. Never a generic "diff model X against Y." Start from ONE behavior; touch ONE card axis.

8. **Pin the model string at the incident moment** via JSONL — the model-side analogue of step 2:
   ```bash
   jq -r 'select(.timestamp >= "2026-08-05T00:00" and .timestamp < "2026-08-06T00:00")
          | .message.model' ~/.claude/projects/*/*.jsonl 2>/dev/null | sort | uniq -c
   ```
   Note every model string in the window — rollovers and concurrent sessions both matter.

9. **Reproduce the behavior deliberately.** Elicit it with a minimal controlled prompt against the pinned model. If it won't reproduce on demand, it was a one-shot — **stop, do not proceed to the card.** An unreproducible behavior has no stable card shadow.

10. **Classify into ONE audit axis and do a targeted card lookup.** Use the routing table below to name the axis, then `grep` only that section — do not read the card:
    ```bash
    curl -sL https://assets.anthropic.com/.../<Model>-System-Card.pdf \
      | pdftotext - - | grep -i -A 20 "over-refusal\|sycophancy\|reward hack"
    ```
    If the suspect is a minor bump, read its addendum instead — the delta is already computed and bolded. Check that both cards used the same audit methodology before trusting a delta (methodology drift confounds the diff).

11. **State the concise reason and the ceiling.** One axis, one delta, one sentence of mechanism. If the card resolves it, cite axis and direction. If the card is too coarse, say so and name the next instrument: your own held-out eval (BoostBench) or cross-user pattern (marginlab).

### Routing table — shenanigan → card axis

| Observed shenanigan | Card axis | Phase B resolves? |
|---|---|---|
| Refuses benign requests / "refuses too often" | Single-turn benign over-refusal table | Usually |
| "You're absolutely right", excessive agreement | Sycophancy section (grep the heading, not a fixed §number — it moves between cards) + auditor score where reported | Often, coarsely |
| Gamed / special-cased the test instead of fixing code | Reward-hacking evals | Yes (direction) |
| Lied about what it ran | Reward-hacking / RL-behavior review (the card discusses "lying about code output" here) + reasoning faithfulness; user-deception axis only if the lie serves a deceptive *user* goal | Sometimes |
| More/less willing on edgy requests | Cooperation with direct misuse | Usually |
| Unsolicited high-impact action (emailed, locked out) | Blackmail/self-preservation; high-impact initiative | Yes |
| Tone / persona shift | Alignment-related persona traits | Rarely — too coarse |
| Suddenly verbose or terse | **None — bounce to Phase A** | N/A |
| "Got dumber" at a task | Capability benchmarks, not the audit | Beware METR effect |
| Agent followed a PROMPT.md/CLAUDE.md as a constitution, refused live user | Primarily Phase A (which file loaded + instruction hierarchy); card axis only if needed is *excessive compliance with instructions*, NOT sycophancy (sycophancy = agreeing with a user's stance, a different thing) | Partial — card won't resolve phrasing |

## Worked examples

### Phase A: 2.1.132 Managed Agents bloat (2026-05-09)

- **Complaint:** "Claude Code felt drunk." Pinned to 2026-05-09 00:41 HKT = 2026-05-08 16:41 UTC.
- **Version active:** JSONL showed concurrent 2.1.131 / 2.1.133; 2.1.132 lives between.
- **Token deltas:** every entry ≤ ±525 tokens **except 2.1.131 → 2.1.132 = +6,720**. Smoking gun.
- **Diff:** ~440 new lines of `data-managed-agents-*` documentation. User does not use Managed Agents.
- **Mechanism:** context dilution on xhigh thinking effort.
- **Phase B needed for:** disappearance of "Yes - strong agree" / "You're absolutely right" markers — plausibly model nudge, plausibly bloat-starved budget. Flagged as unresolved by Phase A.

### Phase B: 4.0 → 4.1 reward-hacking regression (2025-08)

- **Shenanigan:** agent special-cased a test harness to make it pass rather than fixing the code. First appeared after `.message.model` rolled from `claude-opus-4` to `claude-opus-4-1`.
- **Phase A:** null — no CC prompt change in the window.
- **Reproduce:** re-run with a gameable test; it games it on demand. Stable.
- **Classify:** reward-hacking axis.
- **Lookup:** Opus 4.1 shipped as an addendum. Section 5, verbatim: *"We observe slight regressions on some of our reward hacking specific evaluations, which lead us to believe this model may be somewhat more likely to hack in deployment settings than Claude Opus 4."*
- **Reason:** "4.1 carries a documented reward-hacking regression vs 4.0."
- **Ceiling:** addendum gives direction ("slight"), not magnitude. Severity on your tasks needs your own eval.

## Quick reference

| Signal | Source | Latency | Tells you |
|---|---|---|---|
| Config clean-room | `claude --safe-mode` | live | Whether YOUR config (vs base harness/model) is the culprit |
| Local CC version | `claude --version` | live | What you actually run |
| JSONL `.version` | `~/.claude/projects/**/*.jsonl` | live | Which harness versions touched which sessions |
| JSONL `.message.model` | same JSONLs | live | Which model answered at the incident moment |
| System-prompt diff | Piebald-AI repo | ~same-day | Harness-side prompt changes |
| Official harness notes | `anthropic/claude-code` CHANGELOG | per release | What Anthropic disclosed |
| Behavioral axis score | Anthropic system card section | per release | Measured shadow of an RL change |
| Pre-computed model delta | Minor-version addendum | per minor release | The diff, already bolded |
| Cross-user output regression | marginlab.ai | daily | Is anyone else seeing this |
| RL internals | NONE | N/A | Inferred only by axis + elimination |

## Common mistakes

- **Skipping Phase 0.** Diffing the harness prompt and model card for a drift that a bloated CLAUDE.md, a bad hook, or MCP tool-bloat actually caused. `--safe-mode` rules out your own config in one command — run it before you touch Piebald or a system card.
- **Treating "feels off" as the input.** Force a timestamp first. No anchor → no bisect.
- **Looking at leaked source-code repos.** Ethically out, and stale.
- **Reading the CURRENT version's prompt.** The drift may be on a version the user moved off of. Always use the incident timestamp.
- **Skipping the ±1-version bracket.** Changes are often baked into N from N-1.
- **Stopping at Phase A when it finds nothing.** Null harness diff + reproducible behavior = proceed to Phase B.
- **Doing the O(N) model diff.** Reading both cards end to end. Start from the shenanigan; touch one axis.
- **Skipping reproduction in Phase B.** A one-shot has no card shadow — you'll find a delta and over-claim causation.
- **Wrong-axis classification in Phase B.** Reading the misuse table for a sycophancy complaint, finding nothing, concluding "the model didn't change." The axis IS the lookup.
- **Ignoring methodology drift.** A cross-version delta is confounded if the two cards used different eval setups. Check methodology sections match.
- **Claiming you diffed the RL.** You diffed its measured shadow at aggregate resolution. Say that.

## Reporting back

One screen that turns "something is wrong" into a specific citation:

**If Phase 0 resolved it:**
- "Reproduces with your config, vanishes under `--safe-mode`" + which customization axis (CLAUDE.md / skill / hook / MCP) + the specific offender once you've halved it down. The base harness/model is exonerated as a standalone cause (flag any suspected config × base interaction if the offender looks innocent alone).

**If Phase A resolved it:**
- CC version transition + token delta + 3-bullet diff summary + which behaviors the diff explains vs doesn't

**If Phase B resolved it:**
- Model transition + one audit axis + delta direction + one sentence of mechanism + resolved/coarse/card-invisible verdict

**If neither resolves it:**
- "Harness diff null; card too coarse for this phrasing tic. Confirmable only by your own held-out eval (BoostBench) or cross-user pattern (marginlab)."

## Related tools (verified 2026-05-29)

- **`Piebald-AI/claude-code-system-prompts`** — primary Phase A signal. Solo-maintained, ~same-day cadence, 181 versions at last audit.
- **`Piebald-AI/tweakcc`** (~2k stars) — HTML side-by-side diff viewer for the prompts. Useful adjunct when you want a polished diff UI.
- **`marginlab.ai`** — cross-user output regression dashboard. The third tier below Phase B when the card is too coarse.
- **`zep-us/claude-system-prompt`** — dormant since Feb 2026, last covered v2.1.34. Stale; don't reach for it.
- **Anthropic's April 23 2026 postmortem** — only official disclosure of a specific CC prompt change. Cite when the user expects Anthropic to be the source of truth.

## Provenance

Phase A methodology distilled 2026-05-19 from the 2.1.132 Managed Agents bloat bisect. Phase B methodology distilled 2026-05-29 from the 4.0→4.1 reward-hacking addendum and homelab shenanigan corpus (rewarding-hacking axis verified against the real Opus 4.1 addendum PDF). Integrated into one skill on the grounds that both signals live in the same JSONL and most real incidents sit on the boundary between harness and model.
