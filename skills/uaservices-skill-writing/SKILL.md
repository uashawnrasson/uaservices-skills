---
name: uaservices-skill-writing
description: Use when writing a new skill for the UA Services MCP / LUNA session musician workflows. Fires when the user says "capture this as a skill", "write a skill for this", "let's make a skill", or after a session where a non-obvious workflow was discovered and the user wants to preserve it.
---

# Writing UA Services Skills

Skills for this environment fill a specific gap: the agent will always have `get_runtime_helpers`, `get_patch_ops`, and the catalog API (`get_catalog_api`) in context before acting. Assume all of that is already known. A skill only earns its place if it contains something the agent cannot derive from those sources.

## The one question that matters

Before writing anything, ask: **"Would an agent reading the helpers, patch ops, and API schema already know this?"**

If yes — don't write it. It's noise that bloats context and trains the agent to ignore skills.

If no — write exactly that, and nothing else.

## What belongs in a skill

**Non-obvious sequences.** When the correct workflow requires multiple operations in a specific order that isn't apparent from any single helper's docstring. The multi-segment version pattern is the canonical example: `place_version` creates a playlist, then `switch_version`, then `place_generated_clip` for subsequent segments. No individual helper documents this chain.

**Undocumented constraints.** Behaviors the agent would discover only by failing. Example: `place_version` creates a new playlist on every call — calling it twice for two segments of the same take silently creates two separate versions instead of one.

**Producer intent rules.** Hard rules about what the agent should never do without being asked, where the reason is producer workflow rather than technical constraint. Example: don't switch back to V1 after placing takes — the producer owns version state. This isn't in any schema; it's a workflow principle.

**Cross-tool decisions.** When two tools accomplish superficially similar things but the right choice depends on intent. Example: comp lanes (`insert_region` with `layer`) vs versions (`place_version`) — both stack clips, but one is for comping and the other is for auditioning. The schemas describe what each does; a skill describes which to use when.

**Buried efficiency.** When the agent would reach for an extra API call but the data is already in context. Example: version state is in `view_track(doc)['versions']` — no need for a separate `list_versions` call when you already have the export.

## What does not belong in a skill

- Anything documented in the layer-1 docstring of a helper
- Anything in the patch op schema description
- Anything in the catalog API (`get_catalog_api`)
- General patterns the agent would infer from signatures alone
- Warnings that duplicate existing docstring warnings
- Background explanation of how the system works (the agent knows this)

## Format rules

**Name:** Specific to the exact scenario. `luna-multi-segment-version` not `luna-version-management`. The name should fail to trigger on adjacent scenarios.

**Description:** State the precise trigger condition. Include explicit "do NOT use when" if there's a likely confusion with another skill or tool. Trigger phrases should be producer-natural, not technical.

**Body:** Code over prose. One pattern block is worth ten sentences of explanation. No preamble, no summary, no "here's what this skill covers." Start with the thing the agent needs to know.

**Length:** If it's more than 40 lines, question every line. Skills bloat because writers include context that feels important but the agent already has. Cut until it hurts, then check whether anything broke.

## The test

After drafting, read it as if you just called `get_runtime_helpers` and `get_patch_ops`. Cross out every line that is now redundant. If more than half survives, the skill has real value. If less than half survives, the skill is probably noise — consider whether the remaining lines should go in a docstring instead.

## Example: what the trimming looks like

First draft of `luna-multi-segment-version` was 90 lines covering: comp lanes vs versions, V1 check rule, place_generated_clip for take 1, place_version for takes 2+, multi-segment pattern, version state checking, cleanup.

After trimming against what's in helpers/schemas, it became 20 lines covering only: the multi-segment pattern (not in any docstring) and export-based version state (efficiency, not correctness).

Everything else was already in `place_generated_clip` and `place_version` docstrings.
