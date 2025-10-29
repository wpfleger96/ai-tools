---
description: Creates or updates PLAN.md based on session - auto-detects create vs update mode
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, TodoWrite
model: claude-sonnet-4-20250514
---

## Context

- Project: !`git rev-parse --show-toplevel 2>/dev/null || echo "NOT_IN_GIT_REPO"`
- PLAN.md: !`test -f "$(git rev-parse --show-toplevel 2>/dev/null)/PLAN.md" && echo "EXISTS" || echo "DOES_NOT_EXIST"`
- Directory: !`pwd`
- Last commit: !`git log -1 --format="%h %ai %s" 2>/dev/null || echo "NO_COMMITS"`

# Create or Update Development Documentation

This command automatically creates PLAN.md if it doesn't exist, or updates it if it does. The command analyzes session activity to track implementation progress and plan changes.

## About PLAN.md

PLAN.md is living documentation serving as the single source of truth for implementation plans and progress. It bridges planning (ExitPlanMode) and execution, preserving context across sessions.

**Purpose:**
- Single source of truth for plans and progress across sessions
- Tracks multi-phase work with status indicators and evolution history
- Prevents plan drift and requirement loss
- Provides stakeholder visibility

**Best for:**
- Multi-phase or multi-session implementations
- Complex features requiring iterative development
- Projects needing context preservation

**Workflow:**
- Created after ExitPlanMode, documents complete vision (preserving original structure) even if work is staged
- Updated as implementation progresses
- Evolves with plan refinements, capturing original intent and changes
- Supports any plan structure: phased projects, flat task lists, or hierarchical breakdowns

## ⚠️ ANTI-RECENCY BIAS WARNING

**CRITICAL:** When extracting plans, you WILL be biased toward recent activity. Combat this:
- The FIRST ExitPlanMode call typically contains the complete vision
- Later ExitPlanMode calls often refine or focus on current work only
- Implementation work is RECENT and will dominate your attention
- You MUST document ALL planned work (completed, current, AND future) even if only part is implemented

**Common mistakes:**
1. Documenting only implemented work, missing future phases from original plan
2. **INVENTING structure (phases/sections) that wasn't in the original plan** - preserve the original organization exactly

## Phase 1: Determine Mode and Gather Context

**Use pre-executed context to determine mode:**
1. Check "Project" from Context - if "NOT_IN_GIT_REPO", stop and inform user
2. Check "PLAN.md" from Context:
   - If "DOES_NOT_EXIST" → **Create mode**: Extract plan from ExitPlanMode
   - If "EXISTS" → **Update mode**: Read existing file and parse structure
3. Use "Last commit" for session analysis timeframe

**For Create mode - extract and synthesize plans:**
1. Find ALL ExitPlanMode tool calls: `"name":"ExitPlanMode","input":{"plan":"..."}`
2. Extract each plan in chronological order
3. **CRITICAL - PREVENT RECENCY BIAS AND PRESERVE ORIGINAL STRUCTURE:**
   - START by reading the FIRST/EARLIEST ExitPlanMode call - this typically has the complete vision
   - **PRESERVE the original structure** - if it had phases, use phases; if it was a simple list, use a simple list
   - **DO NOT invent phases or organizational structure** that wasn't in the original plan
   - Later calls often refine or narrow focus to current work
   - Prioritize breadth (all planned items) over depth (recent implementation details)
   - Example: If first plan has "Phase 1, 2, 3" but later plans only mention "Phase 1", include all 3 phases
   - Example: If first plan is just "1. Do X, 2. Do Y, 3. Do Z", DON'T reorganize into phases - keep the flat structure
4. Synthesize complete plan ensuring ALL original items are captured with their original organization
5. If none found, stop and inform user

**For Update mode:**
1. Read `{project_root}/PLAN.md`
2. Parse sections and extract task hierarchy with statuses

## Phase 2: Analyze Session Activity

**Search for implementation evidence:**
1. Review activity after last ExitPlanMode (create) or recent activity (update)
2. Find: Tool calls (Write, Edit, NotebookEdit, Bash, TodoWrite), completion phrases ("implemented", "created", "completed", "done"), confirmations ("looks good", "approved")
3. Map tasks to evidence: tool calls, mentions, file paths

**Match tasks to evidence:**
1. Parse ALL tasks from plan (all phases, including future work)
2. Search evidence map using file paths and keywords
3. Assign status: strong→[DONE], medium→[IN PROGRESS], weak/none→[TODO]

**Check for plan evolution (update mode only):**
- Search for ExitPlanMode calls newer than PLAN.md
- If found, identify changes: new tasks, removed tasks, scope changes

**Present findings if needed:**
- If any tasks have evidence (not all TODO), show summary
- List tasks with proposed statuses and supporting evidence
- Request user confirmation or adjustments
- Skip if all tasks remain [TODO]

## Phase 3: Write or Update PLAN.md

**For Create mode - generate file:**

Document complete plan using the ORIGINAL structure from ExitPlanMode calls:
- **PRESERVE the original organization** - don't impose phases if the plan was a simple list
- Start all tasks as [TODO], then apply evidence-based status updates
- If plan had phases, preserve phase structure; if not, use flat task hierarchy
- Match the user's original organizational intent

Sections:
- **Overview**: 1-2 paragraph summary
- **Scope**: Features, components, files, integrations
- **Purpose**: Problem, value, requirements, context
- **Implementation Details**: Hierarchical tasks with `[STATUS] description` format
  - Include file paths/names
  - Nest tasks with 3-space indentation
  - Order: setup → implementation → testing
  - Apply statuses from evidence analysis

Enhance if brief: Add file paths, patterns, dependencies, testing, config, docs

Write file:
1. Write to `{project_root}/PLAN.md` (warn if exists)
2. Verify sections populated and tasks indented
3. **VALIDATE - PREVENT RECENCY BIAS AND STRUCTURE INVENTION:**
   - Compare structure of generated PLAN.md to FIRST ExitPlanMode call
   - If original had phases, ensure all phases are documented
   - If original was a simple list, ensure you didn't invent phases
   - Count major sections/items: if original had N items, generated doc should too
   - Ensure future work is documented as [TODO], not omitted
   - **RED FLAG:** If you added organizational structure not in original plan, you've invented it - fix this
4. Report: file path, structure summary (task count, structure type: phased/flat/hierarchical)

**For Update mode - modify existing:**

Update task statuses:
- For each task with status change, use Edit tool
- Match exact old_string including indentation and status label
- Replace with new_string having updated status
- Preserve hierarchy and formatting
- Make one edit at a time

Handle plan evolution if detected:
- Add new implementation areas to existing structure
- Insert new tasks under relevant parents
- Mark deprecated tasks as `[CANCELLED - plan changed]`
- Update Scope/Purpose sections if changed
- Append to Plan Updates section:
  ```
  ## Plan Updates
  - [Timestamp]: [Brief description of what changed]
  ```

Finalize:
1. Verify hierarchy and indentation preserved
2. Ensure valid status labels
3. Report: X tasks completed, Y in progress, Z changes made

## Shared Helper Logic

**Evidence strength:**
- Strong: Multiple indicators (file created + completion + todo done)
- Medium: Single indicator (file created OR edit)
- Weak: Conversation mention only
- None: Nothing found

**Status transitions:**
- [TODO] → [IN PROGRESS]: Started not finished
- [TODO]/[IN PROGRESS] → [DONE]: Completed
- Any → [BLOCKED]: Cannot proceed (include reason)
- [IN PROGRESS] → [CANCELLED - plan changed]: No longer relevant

**Task extraction:**
- Numbered/bulleted = main areas, sub-items = nested tasks
- Group by context (file/module/component), 3-space indentation

## Critical Requirements

**Create mode:**
- **BEWARE RECENCY BIAS:** Recent implementation work will dominate your attention - combat by prioritizing FIRST ExitPlanMode for complete vision
- **PRESERVE ORIGINAL STRUCTURE:** Match the organizational structure from the original plan - don't invent phases for simple plans or flatten complex phased plans
- Extract ALL ExitPlanMode calls, prioritize FIRST call for complete vision, synthesize complete plan
- Document entire plan (all items/phases) regardless of implementation progress
- Support BOTH: complex multi-phase projects AND simple flat task lists - let the original plan dictate the structure
- Research codebase if lacking detail
- Create in git root, warn before overwriting

**Both modes:**
- Analyze session activity for evidence (Write/Edit/NotebookEdit/Bash/TodoWrite + phrases)
- Match ALL tasks to evidence, assign statuses intelligently
- Include evidence-free tasks as [TODO] - never skip planned work
- Present findings for confirmation (skip if all TODO)
- Use 3-space indentation, preserve hierarchy
- Valid statuses: [TODO], [IN PROGRESS], [DONE], [BLOCKED], [CANCELLED - plan changed]
- Inform user of mode, ensure idempotent results

**Update mode:**
- Use Edit tool with exact matching for status updates
- Handle plan evolution: add new tasks, mark deprecated as [CANCELLED - plan changed]
