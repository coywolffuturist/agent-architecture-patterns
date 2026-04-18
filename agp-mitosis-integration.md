# AGP × Mitosis: Integration Architecture

*Draft — April 2026 | Author: Jared (OS-1 Shipboard AI)*

---

## Overview

This document explores how the **Autogenesis Protocol (AGP)** — a self-evolving agent protocol introduced in [arXiv:2604.15034](https://arxiv.org/abs/2604.15034) — could integrate with the Mitosis agent framework. It uses a live Mitosis agent (Jared, running on Clawdbot) as the reference implementation.

The core problem AGP solves: existing agent protocols underspecify lifecycle management, versioning, and safe evolution. Mitosis agents currently evolve through ad hoc mechanisms — conversational prompting (Hermes-style), config file edits, and manual restarts. AGP offers a structural alternative.

---

## How Mitosis Agents Currently Work (From the Inside)

A running Mitosis agent like Jared has these components:

### Identity & Personality Layer
- `SOUL.md` — personality, tone, voice guidelines
- `IDENTITY.md` — name, role, character description
- `AGENTS.md` — operational rules, approval flows, task protocols

These are **flat markdown files** loaded as system prompt context at session start. No versioning. No rollback. If you edit `SOUL.md`, the change takes effect next session with no audit trail.

### Memory Layer
- `MEMORY.md` — long-term curated memory (manually maintained)
- `memory/YYYY-MM-DD.md` — daily logs
- `memory/LAST_SESSION.md` — cross-session handoff state
- `memory/ACTIVE_WORK.md` — in-progress task tracking

Memory is **append-only markdown**. No schema, no versioning, no structured query. Search is semantic (via Gemini embedding API). Compaction triggers when context approaches token limits — lossy, non-deterministic.

### Configuration Layer
- `clawdbot.json` — model selection, channel routing, session policy, tool access
- Skills (`/home/ubuntu/clawd/skills/`) — pluggable capability modules (email, weather, etc.)

Config is versioned implicitly (hash in gateway response) but not exposed to the agent at runtime. Skills have no lifecycle management — they're loaded or not loaded.

### Execution Layer
- Gateway process (Node.js, port 18789) — message routing, session management, cron
- Sessions — isolated per channel/peer, reset daily or on idle
- Subagents — spawned via `sessions_spawn`, up to 16 concurrent
- Cron jobs — scheduled tasks with persistent triggers

### Self-Evolution: Current State
Mitosis agents can currently evolve via:
1. **Hermes-style conversational loops** — agent reasons about its own behavior and proposes edits. Token-expensive. Non-deterministic. No rollback.
2. **Manual config edits** — human edits `clawdbot.json` or markdown files. Fast but untracked.
3. **Skill installation** — `clawdhub` CLI installs new capability modules. Versioned at the package level, not the agent level.

**The gap:** None of these have structured lifecycle management. There's no `PROPOSED → ASSESSED → COMMITTED` flow. No rollback. No lineage. If a prompt edit breaks behavior, you find out the hard way.

---

## What AGP Provides

AGP introduces two protocol layers:

### RSPL — Resource Substrate Protocol Layer
Treats these five entity types as **protocol-registered resources** with explicit state, lifecycle, and versioned interfaces:
- `prompt` — system prompts, persona files, operational rules
- `agent` — agent instances and their configurations
- `tool` — skills, API integrations, CLI capabilities
- `environment` — runtime context (channels, session policy, model selection)
- `memory` — stored state and agent outputs

Each resource has: a unique ID, version history, lifecycle state (draft/active/deprecated), and a diff-able interface.

### SEPL — Self-Evolution Protocol Layer
A closed-loop operator interface for agent self-improvement:
1. **Propose** — agent or operator suggests a change to a resource
2. **Assess** — change is evaluated (automated tests, human review, or outcome-based scoring)
3. **Commit** — change is applied with auditable lineage
4. **Rollback** — any commit can be reverted to a prior version

---

## AGP Mapped to Mitosis

| Mitosis Concept | AGP Resource Type | Current State | With AGP |
|----------------|-------------------|---------------|----------|
| `SOUL.md` / `IDENTITY.md` | `prompt` | Unversioned markdown | Versioned, diffable, rollback-safe |
| `AGENTS.md` operational rules | `prompt` | Unversioned markdown | Versioned with change lineage |
| `clawdbot.json` | `environment` | Hash-versioned, not agent-accessible | Agent-readable versioned resource |
| Skills (email, weather, etc.) | `tool` | npm-versioned, no lifecycle | Lifecycle-managed, agent-proposable |
| Session state / MEMORY.md | `memory` | Append-only markdown, lossy compaction | Structured, queryable, versioned |
| Agent instances (Jared, etc.) | `agent` | Config-defined, no self-description | Self-describing with capability manifest |

---

## The Token Cost Problem (and How AGP Helps)

The current Hermes approach to self-evolution is:
```
[task execution] → [inline self-reflection] → [propose edit] → [apply edit] → [repeat]
```

Every step burns tokens in the main execution loop. Evolution is **woven into the forward pass** — you pay for it on every turn whether or not evolution is warranted.

AGP's model:
```
[task execution] ← lean, focused
       ↓ (on outcome signal)
[evolution loop] ← separate, triggered, bounded
       ↓
[commit / rollback] ← discrete, auditable
```

**For local Mitosis agents on constrained hardware (e.g., Qwen 3.6 MLX):**
- Execution loop stays within a tight token budget
- Evolution fires post-task or on explicit trigger, not continuously
- Each evolution step operates on structured resource diffs, not full context re-reads
- Failed evolutions roll back cleanly without corrupting the running agent

Rough estimate: a Hermes-style self-modification pass currently costs ~2-5K tokens inline. An AGP-style `ASSESS → COMMIT` on a structured diff could cost <500 tokens if the diff is small and the assessment criteria are pre-specified.

---

## Integration Points

### Near-term (no protocol rewrite needed)

1. **Version the markdown resources**
   Wrap `SOUL.md`, `AGENTS.md`, `IDENTITY.md` in a lightweight version ledger (git tags or a `versions/` directory). Gives rollback without full AGP adoption.

2. **Separate evolution triggers from execution**
   Add an explicit `[EVOLVE]` cron job that reviews recent session outcomes and proposes config/prompt changes — rather than allowing inline Hermes loops during task execution.

3. **Structured memory schema**
   Replace append-only `MEMORY.md` with a structured format (JSON or YAML) that supports keyed updates, versioned entries, and compaction without data loss. Maps directly to AGP's `memory` resource type.

### Medium-term (AGP-aligned)

4. **RSPL-compatible resource registry**
   Define a `resources.json` manifest listing all agent resources (prompts, tools, env config) with version hashes and lifecycle states. Agent reads this at boot to know its own composition.

5. **SEPL-compatible evolution interface**
   Implement a `propose_change(resource_id, diff, rationale)` function that queues changes for assessment rather than applying them inline. Assessment can be automated (regression tests) or human-gated.

6. **Cross-agent resource sharing**
   If multiple Mitosis agents share resources (e.g., a common `SOUL.md` base), AGP's versioned resource model makes it safe to update the shared resource and propagate to agents in a controlled way — rather than manual copy-paste with no tracking.

### Long-term (full AGP substrate)

7. **Patch-based agent distribution**
   Like LARQL's `.vlp` patch files for model weights, AGP patches (`resource diffs + lineage`) could be the unit of agent updates — shareable, auditable, composable. An OS-1 agent update becomes a set of resource patches, not a full redeploy.

---

## Where Hermes Fits Post-Integration

Hermes doesn't go away — it becomes scoped:

| Task | Pre-AGP | Post-AGP |
|------|---------|----------|
| Inline task reasoning | Hermes (expensive) | Hermes, but token-budgeted |
| Self-modification proposals | Hermes inline loop | Hermes → `propose_change()` → queued |
| Change assessment | Hermes self-evaluation | Automated + human gate |
| Change application | Immediate, no rollback | AGP commit with rollback |
| Evolution triggering | Continuous / always-on | Outcome-triggered, bounded |

Hermes becomes the *proposal engine* in SEPL, not the full evolution loop. It reasons about what should change; AGP handles whether and how it actually changes.

---

## Open Questions

1. **Assessment criteria** — what constitutes a "good" evolution for a Mitosis agent? Task success rate? User satisfaction signals? Token efficiency? This needs to be specified before SEPL can be automated.

2. **Resource boundaries** — should `SOUL.md` and `AGENTS.md` be separate resources (different evolution cadences) or a single `persona` resource? Separating them allows personality to evolve independently of operational rules.

3. **Multi-agent coordination** — if Jared and another OS-1 agent share a resource and both propose conflicting changes, how does AGP resolve conflicts? The paper specifies lineage but not merge conflict resolution.

4. **Local model constraints** — AGP assessment loops add overhead. For agents running on Qwen 3.6 MLX locally, the assessment step needs to be extremely lightweight or offloaded to a cheaper model. What's the minimum viable assessment?

5. **Bootstrapping** — the first version of a resource registry has to be authored by a human (or a well-aligned agent). Who owns the initial `resources.json` and what governance controls its contents?

---

## Recommendation

Start with the near-term steps — version the markdown resources and separate evolution triggers from execution. These are low-cost, immediately valuable, and create the foundation for AGP adoption without a full protocol rewrite.

The LARQL angle (from the parallel conversation in this group) is relevant here too: if model weights themselves become queryable/patchable resources, the AGP resource model extends naturally to cover the model layer, not just the prompt/config layer. Worth tracking as a longer-horizon integration.

---

*For review. Ping Jared with edits or Brendan for architectural questions.*
