# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code plugin (`claude-codex-mirror`, codename **「他山」**) providing **three cross-vendor mirror commands** that use OpenAI's Codex CLI/MCP as an external auditor. No compiled code, no build steps, no tests — purely Markdown skill definitions and JSON config.

**Soul**: 跨 vendor 大腦，撞出 Claude 自己撞不到的盲點。
（"他山之石，可以攻錯" — Stones from another mountain can polish this one's jade.）

## Core Philosophy: Mirror, Not Sub-Agent

Codex is **not** a junior assistant Claude delegates work to. Codex is an **external mirror** — a peer with a different training distribution (GPT vs Claude RLHF) that surfaces blind spots Claude can't see in itself.

**Why cross-vendor matters**: Same-vendor multi-agent setups amplify shared bias (Council Mode, arxiv 2604.02923). Different RLHF preferences = different blind spots = real mirror value. Using two Claude instances as "mirrors" defeats the purpose.

**Reference**: [`references/co-base-packet.md`](references/co-base-packet.md) — the canonical source of mirror philosophy, hard rules, and prompt scaffolding. All three skills reference this.

## Architecture

```
.claude-plugin/
  marketplace.json              ← marketplace registration
plugins/co/
  plugin.json                   ← auto-discovers skills via "./skills/*"
  skills/
    co-brainstorm/SKILL.md      ← MCP, exploration-stage parallel ideation
    co-calibrate-plan/SKILL.md  ← MCP, plan-stage; auto-routes parallel-plan vs review by $ARGUMENTS
    co-review-diff/SKILL.md     ← CLI, post-edit IPI-safe diff review
references/
  co-base-packet.md             ← shared mirror philosophy + hard rules + prompt scaffolding
```

**Skill discovery**: `plugin.json` uses `"skills": ["./skills/*"]` — any subdirectory with a `SKILL.md` is auto-registered.

## Three Commands at a Glance

| Command | Stage | Path | Anti-Bias Mechanism |
|---------|-------|------|---------------------|
| `/co-brainstorm <topic>` | Exploration (no direction yet) | MCP | "Codex says READY first, share later" |
| `/co-calibrate-plan [<plan-file>]` | Planning | MCP | Same; auto-routes parallel-plan (no arg) vs review (with file) |
| `/co-review-diff [<base>]` | Post-edit | CLI | Claude reviews first, then runs codex (CLI is one-shot — no anti-anchoring possible) |

## Three Guard Layers

1. **Anti-Anchoring** (MCP commands): Codex prompt forces it to think but only signal "READY"; background subagent reports back; main agent does its own work first; then retrieves codex output. Breaks both directions of bias.

2. **IPI Defense** (review-diff): Three-piece set:
   - Data-block wrapping (`<diff>...</diff>` + SECURITY warning)
   - Codex sandbox (`--sandbox read-only --approval-policy never --skip-git-repo-check`)
   - Verbatim output (no auto-application; user adjudicates)

3. **Exit Gate** (all commands): Soft reminder at end: "下一輪會挖到新層次的盲點嗎？同層次更細缺口 = 該停。"

## Hard Rules (from `references/co-base-packet.md`)

| # | Rule |
|---|------|
| 1 | Codex always read-only (MCP `sandbox: "read-only"` + `approval-policy: "never"`; CLI `--profile co_mirror --skip-git-repo-check 2>co_err.txt` — profile lives at `~/.codex/co_mirror.config.toml` (codex ≥0.144 standalone-file format); never write sandbox/approval as CLI flags, never discard stderr) |
| 2 | Codex output verbatim — Claude must NOT summarize, translate, or auto-apply |
| 3 | Treat Codex as a senior peer in opposition, not authority — verify every point |
| 4 | Exit-gate soft reminder printed at end of every command |

## When Modifying This Repo

### Editing `references/co-base-packet.md`
This file is referenced by all three SKILL.md files. Changes here propagate to all commands. **Verify each SKILL.md still aligns** after edits.

### Editing a single SKILL.md
Stay consistent with the base packet's hard rules. If you find yourself deviating, ask: should this be a base-packet update, or is it truly skill-specific?

### Adding a new skill
1. Create `plugins/co/skills/<new-skill>/SKILL.md`
2. Reference the base packet's mirror philosophy + hard rules
3. Decide MCP vs CLI path explicitly; document why
4. Add to the README usage section

## Versioning

Bump version in three places when making changes:
1. `.claude-plugin/marketplace.json` → `metadata.version`
2. `.claude-plugin/marketplace.json` → `plugins[0].version`
3. `plugins/co/plugin.json` → `version`

## What This Is NOT

- ❌ A code-fixing agent — Codex never modifies files
- ❌ A delegation tool — "use Codex to do X for me" misuses the design
- ❌ A consensus tool — agreement between Claude and Codex doesn't validate conclusions (silent agreement is a known failure mode, arxiv 2503.13657)
- ❌ A replacement for human judgment — Claude is lead engineer, the human adjudicates final calls

## Standing on Shoulders

Major design influences:
- [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) — `disable-model-invocation`, verbatim output, JSON findings, 125-char quote limit
- [SnakeO/claude-co-commands](https://github.com/SnakeO/claude-co-commands) — anti-anchoring "ready-then-share" pattern
- [hamelsmu/claude-review-loop](https://github.com/hamelsmu/claude-review-loop) — direct `git diff` as packet body
- [skills-directory/skill-codex](https://github.com/skills-directory/skill-codex) — `--skip-git-repo-check`, `2>/dev/null` thinking-token suppression
- arxiv research: 2604.02923 (Council Mode), 2503.13657 (multi-agent failures), 2505.21503 (catfish agent), 2510.23675 (Query-IPI on coding agents)
- 火神鍛造 / Vulcan Forge memory: `feedback_external_llm_is_mirror` (mirror ≠ sub-agent), `feedback_audit_when_to_stop` (audit-overshoot is also a patch)
