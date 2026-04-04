# Multi-Agent E2E Testing Implementation Plan

> **For agentic workers:** This plan modifies SKILL.md only. No application code. Read the design spec at `docs/superpowers/specs/2026-04-04-multi-agent-e2e-design.md` first.

**Goal:** Add orchestrator mode to CrowdTest SKILL.md — auto-detect multi-agent capability, dispatch sub-agents for each persona with real Playwright E2E testing, collect results, maintain backward compatibility.

**Architecture:** SKILL.md gains an "Execution Mode Detection" section and an "[ORCHESTRATOR]" alternate path for Phases 4-6. Legacy mode is preserved with `[LEGACY]` tags. Sub-agents receive self-contained PERSONA-TASK.md instructions and output standardized JSON.

**Tech Stack:** SKILL.md (prompt-as-code), Playwright MCP, OpenClaw sessions_spawn / Codex CLI / Claude Code CLI

---

## Task 1: Add Execution Mode Detection Section

**Files:**
- Modify: `SKILL.md` (insert after "## Prerequisites" section, before "## How It Works")

- [ ] **Step 1: Write the Execution Mode Detection section**

Insert a new `## Execution Mode Detection` section that instructs the agent to:
1. Check if `sessions_spawn` tool is available → Orchestrator Mode (OpenClaw)
2. Else check if `codex` CLI exists → Orchestrator Mode (Codex dispatch)
3. Else check if `claude` CLI exists → Orchestrator Mode (CC dispatch)
4. Else → Legacy Mode (current behavior)
5. Allow `--mode orchestrator|legacy` flag to override auto-detection
6. Print which mode was detected before proceeding

- [ ] **Step 2: Add mode tags to "How It Works" overview**

Update the pipeline diagram to show both paths:
- `[ORCHESTRATOR]` Phase 4-6: Sub-agent dispatch per persona
- `[LEGACY]` Phase 4-6: Single agent serial execution (current)

- [ ] **Step 3: Verify no existing content is broken**

Read through the section flow to ensure the new section integrates cleanly.

---

## Task 2: Write Sub-Agent Persona Test Package Template

**Files:**
- Modify: `SKILL.md` (insert new section before the `[ORCHESTRATOR]` Phase 4)

- [ ] **Step 1: Write the "Persona Test Package" section**

Define the complete `PERSONA-TASK.md` template that the main agent generates for each sub-agent. Must include:
- Persona identity (name, age, archetype, tech_level, behavioral_rules)
- Task assignment (primary task, success criteria, secondary tasks)
- Target product info (URL, SiteMap JSON, ProductProfile summary)
- Already known issues (covered_issues from previous personas)
- Browser setup instructions (Playwright, viewport by device type)
- Testing loop instructions (ORIENT → ACT → OBSERVE → DECIDE)
- Action recording format (JSON schema for action-log)
- Emotional tracking instructions
- Circuit breakers (max 20 actions, max 3 min, 5 consecutive failures)
- Output file requirements (action-log.json, journey.json, feedback.json)

- [ ] **Step 2: Write output JSON schemas**

Define the exact JSON schemas for:
- `action-log.json` — raw action log with steps, emotional states, confusion events
- `journey.json` — confidence curve, key moments, narrative
- `feedback.json` — issues, scores, positives, verdict

These schemas must be compatible with Phase 7 aggregation (existing code).

---

## Task 3: Rewrite Phase 4-6 with Orchestrator Path

**Files:**
- Modify: `SKILL.md` (Phase 4, Multi-Persona Orchestration, Phase 5, Phase 6 sections)

- [ ] **Step 1: Tag existing Phase 4 content as [LEGACY]**

Wrap the existing Phase 4: Browser Sessions content with a clear `[LEGACY]` marker:
```
### [LEGACY MODE] Phase 4: Browser Sessions
(existing content unchanged)
```

- [ ] **Step 2: Write [ORCHESTRATOR] Phase 4: Sub-Agent Browser Sessions**

New section covering:
1. Worktree creation for each persona (`/tmp/crowdtest-persona-N`)
2. Writing PERSONA-TASK.md to the worktree
3. Dispatching via sessions_spawn (OpenClaw) or exec (Codex/CC CLI)
4. Waiting for completion (polling worktree for output files)
5. Reading output JSON files
6. Error handling (timeout, crash, missing files)
7. Worktree cleanup
8. Passing covered_issues to next persona

- [ ] **Step 3: Tag existing Multi-Persona Orchestration as [LEGACY]**

The existing orchestration loop runs Phase 4→5→6 inline. Tag it as `[LEGACY]`.

- [ ] **Step 4: Write [ORCHESTRATOR] Multi-Persona Orchestration**

New orchestration loop that:
1. Creates worktree per persona
2. Writes PERSONA-TASK.md
3. Dispatches sub-agent
4. Waits for completion
5. Reads action-log.json, journey.json, feedback.json from worktree
6. Reports progress
7. Accumulates covered_issues
8. Handles failures (skip, don't abort)
9. Cleans up worktree
10. After all personas: passes results to Phase 7

- [ ] **Step 5: Tag existing Phase 5 inline logic as [LEGACY]**

- [ ] **Step 6: Write [ORCHESTRATOR] Phase 5 note**

In orchestrator mode, Phase 5 (Journey Reconstruction) is done BY the sub-agent as part of its task. The main agent just reads the journey.json output. Add a brief note.

- [ ] **Step 7: Tag existing Phase 6 inline logic as [LEGACY]**

- [ ] **Step 8: Write [ORCHESTRATOR] Phase 6 note**

Same as Phase 5 — sub-agent does feedback synthesis. Main agent reads feedback.json.

---

## Task 4: Update Phase 7-8 to Handle Both Input Formats

**Files:**
- Modify: `SKILL.md` (Phase 7 section)

- [ ] **Step 1: Add input format note to Phase 7**

Phase 7 currently assumes inline data. Add a note:
- `[LEGACY]`: Data is in memory from inline Phase 4-6 execution
- `[ORCHESTRATOR]`: Data is read from JSON files in worktrees (action-log.json, journey.json, feedback.json for each persona)
- Both produce the same data structures, so aggregation logic is identical

- [ ] **Step 2: Verify Phase 7 schema compatibility**

Ensure the output schemas defined in Task 2 match what Phase 7 expects as input. Cross-reference field names.

---

## Task 5: Update Configuration and Cleanup Sections

**Files:**
- Modify: `SKILL.md` (Configuration table, Cleanup section)

- [ ] **Step 1: Add `mode` to Configuration table**

| Option | Default | Description |
|--------|---------|-------------|
| `mode` | auto | Execution mode: `auto` (detect), `orchestrator`, `legacy` |

- [ ] **Step 2: Update Cleanup section**

In orchestrator mode, cleanup also includes:
- Removing temporary worktrees
- Cleaning up sub-agent artifacts

- [ ] **Step 3: Update Cost Estimate table**

Orchestrator mode uses sub-agent tokens (Codex/CC) rather than the main agent's tokens. Add a note about cost differences.

---

## Task 6: Self-Review and Validation

- [ ] **Step 1: Read the complete updated SKILL.md top to bottom**

Verify:
- Mode detection section is clear and complete
- [LEGACY] and [ORCHESTRATOR] paths are clearly distinguished
- Sub-agent package template is self-contained
- Output schemas match Phase 7 input expectations
- No orphaned references or broken flow
- Backward compatibility: removing all [ORCHESTRATOR] content leaves original SKILL.md intact

- [ ] **Step 2: Verify the SKILL.md is parseable as a skill**

Check:
- YAML frontmatter is intact
- All sections have proper heading hierarchy
- No broken markdown formatting

- [ ] **Step 3: Commit the final SKILL.md**

```bash
git add SKILL.md
git commit -m "feat: add multi-agent orchestrator mode with E2E Playwright testing"
```
