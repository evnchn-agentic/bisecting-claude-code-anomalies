# bisecting-claude-code-anomalies

A [Claude Code skill](https://agentskills.io/specification) that teaches an agent to investigate "Claude Code feels different lately" reports — turning subjective vibe drift into a specific citation, whether the cause is harness-side (system prompt, tools, CHANGELOG) or model-side (training nudges, sycophancy, reward-hacking, refusal style).

## What it does

Both signals live in the same JSONL: `.version` for the harness, `.message.model` for the model. The skill runs them in sequence.

**Phase A — harness side** orchestrates four ground-truth sources:

1. **Local JSONL `.version` field** — which CC version was active at the incident moment.
2. **[Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)** — community-extracted system prompts, ~same-day per release, commit titles include `(+N tokens)` deltas.
3. **`anthropics/claude-code` CHANGELOG.md** — official per-version notes.
4. **`claude --version`** — what's installed now.

**Phase B — model side** kicks in when Phase A finds nothing or leaves behavior unexplained:

5. **Local JSONL `.message.model` field** — which model string answered at the incident moment.
6. **Anthropic system cards** — per-model behavioral audits covering over-refusal, sycophancy, reward-hacking, cooperation-with-misuse, and more. Minor bumps ship addenda that pre-diff and bold the better number.

Phase B is O(k) — shenanigan-first. Start from one observed behavior, classify to one audit axis, look up only that section. Never a generic two-card diff.

Pure shell — no bundled scripts, no required state directory, no network telemetry. Each step is one or two commands you can run independently.

## Install

```bash
cd ~/.claude/skills
git clone https://github.com/evnchn-agentic/bisecting-claude-code-anomalies.git
```

The skill is then auto-discoverable by Claude Code in any session — its description triggers on phrases like "feels different," "drunk," "less sharp," "got worse," "got dumber," "degradation," "started doing X yesterday," "no model changes on my end," and similar vibe-drift complaints. Open `SKILL.md` to read or modify.

## Why it exists

Anthropic does not publish per-version Claude Code system prompts. The only official disclosure of a specific prompt change is the [April 23 2026 postmortem](https://www.anthropic.com/engineering/april-23-postmortem), which named one verbosity rule. Community discussion of "Claude got worse" recurs on HN and Reddit but typically ends at "downgrade or wait" rather than at "here is the +6,720-token commit that changed the prompt."

This skill closes that loop on the harness side. It will not solve model-side training drift — it tells you when that's what's happening so you don't over-claim.

## Worked example: 2.1.132 Managed Agents bloat

Documented inline in `SKILL.md`. Complaint pinned to 2026-05-09 00:41 HKT → version-active scan showed concurrent 2.1.131 / 2.1.133 sessions → Piebald commits in window all ≤ ±525 tokens **except 2.1.131 → 2.1.132 = +6,720 tokens** → ~440 lines of `data-managed-agents-*` documentation → context dilution on xhigh thinking effort. The skill also flags what the diff couldn't explain (disappearance of "Yes - strong agree" stylistic markers — plausibly Anthropic-side, plausibly bloat-starved stylistic budget; not bisected).

## Related public tools

- **[Piebald-AI/tweakcc](https://github.com/Piebald-AI/tweakcc)** — same maintainer; HTML side-by-side diff viewer for the system prompts. Aimed at conflict-resolution for users who patch the prompt locally, not at behavior attribution. Useful adjunct if you want a polished diff UI.
- **[marginlab.ai](https://marginlab.ai)** — external daily-benchmark dashboard tracking output-side regressions across LLM versions. Complement, not competitor: measures *outputs*, this skill inspects *prompts*.

## Provenance

Distilled from the 2.1.132 incident bisect (2026-05-19), offered to Falko Schindler and Rodja Trappe (NiceGUI / Zauberzeug) on 2026-05-20 as the response surface for Claude Code anomalies during the Zauberzeug collaboration window. Generalizable to any Claude Code user reporting vibe drift.

## License

MIT
