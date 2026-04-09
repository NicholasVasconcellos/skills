---
name: add-task
description: >
  Add a new task to an existing planToTasks workspace. Use this skill whenever the user
  wants to add a task, create a new task, append a task, or extend the task list in a
  project that already has a tasks/ directory and manifest.json. Trigger phrases include
  "add a task", "new task", "I need another task for...", "create task for...",
  "add a step to the project", or any request to extend the existing task breakdown
  with new work. Also trigger when the user describes work that needs to be done and
  the project already has a planToTasks workspace set up.
disable-model-invocation: true
---

# newTask

Add a single new task to an existing planToTasks workspace, respecting the project's conventions and structure.

## Input

A description of what the new task should accomplish, provided by the user in natural language. The project must already have a `tasks/` directory, `tasks/manifest.json`, and `PROJECT.md` (i.e., `planToTasks` has already been run).

## Execution Steps

### 1. Load context

- Read `PROJECT.md` for tech stack, structure, and conventions.
- Read `progress.txt` for any relevant context about current project state
- Read `tasks/manifest.json` to get the full list of existing tasks.
- Scan `tasks/` to confirm the highest existing task number.

### 2. Determine the next task number

Find the highest-numbered `.md` file in `tasks/` (ignoring `manifest.json` and non-task files). The new task number is that value + 1, zero-padded to 3 digits.

```bash
NEXT=$(ls tasks/[0-9]*.md 2>/dev/null | sed 's|tasks/||;s|\.md||' | sort -n | tail -1 || true)
NEXT=$(printf "%03d" $(( 10#${NEXT:-0} + 1 )))
```

### 3. Identify dependencies

Based on the user's description and the existing manifest, determine which tasks the new task depends on. Use engineering judgment:

- If the new task builds on specific existing work, list those task IDs in `depends_on`.
- If it's independent of everything, use an empty array.
- If unsure, depend on the last task in the manifest as a safe default — sequential is safer than broken parallelism.
- **Ask the user** if the dependency is ambiguous and could go either way.

### 4. Write the task file

Create `tasks/NNN.md` following the task-writing guidelines below.

#### Task splitting — think before writing

Think like a senior engineer. If the user's description covers a lot of ground, consider whether it should be one task or multiple. Apply these heuristics:

**Keep it as one task** when the work is cohesive and an agent can complete it in one session. A single task that sets up a DB schema, seed script, and types is better than three separate tasks — less overhead, fewer handoffs.

**Suggest splitting into multiple tasks** when:

- The deliverable has many unrelated files or subsystems
- Completing everything in one pass would require holding too much context
- There are natural breakpoints where one piece must be verified before continuing
- Later work depends on earlier output (e.g., can't build API routes before the DB exists)

If splitting is warranted, tell the user and propose the breakdown before creating anything.

**Be mindful of context efficiency.** If two tasks would require the agent to read the same set of files, consider merging them.

#### Task file format

Use this template exactly:

```markdown
# Task NNN: [Short descriptive title]

## Context

[2-3 sentences max. What does this task connect to? What should exist before this runs? What will use its output?]

## Files to Read

- PROJECT.md
- [exact/path/to/file.ext — every file the agent should read before starting]

## Files to Write

- [exact/path/to/file.ext — every file the agent will create or modify]
- tests/NNN_short_name.sh

## Steps

1. [High-level step — what to do, not how to do it]
2. [Another step]
3. [...]
   N. Write test script to `tests/NNN_short_name.sh` based on the test suggestions below.

## Test Suggestions

- [What to verify — e.g., "Server starts without errors on port 3000"]
- [Another verification — e.g., "POST /api/users returns 201 with valid payload"]
- [Another — e.g., "TypeScript compiles with no errors"]

## Definition of Done

[Plain English summary of what "complete" looks like. The agent checks this before marking the task done.]
```

#### Task file rules

- **Keep each file under ~100 lines.** If you're going over, the task is too big — split it.
- **Steps stay high-level.** List _what_ needs to happen, not _how_. The executing agent is a capable engineer. "Create the auth middleware" is good. "Import express, create a function called authMiddleware that takes req, res, next..." is too detailed.
- **Files to Read uses exact paths from project root.** No directories, no glob patterns. `src/db/schema.ts` not `src/db/`. Include `PROJECT.md` in every task.
- **Files to Write includes the test script.** The agent writes the test as part of the task. Follow the naming convention: `tests/NNN_short_name.sh`.
- **Test Suggestions are high-level.** Tell the agent _what_ to verify, not _how_. The agent writes the actual test script.
- **Self-contained.** An agent should be able to read one task file + PROJECT.md and know exactly what to do without reading other task files. The Context section handles cross-references.
- **No ambiguity.** If a step could be interpreted two ways, add a brief clarification. But don't over-specify.

### 5. Update manifest.json

Add the new task entry to the `tasks` array in `tasks/manifest.json`:

```json
{ "id": "NNN", "title": "Short descriptive title", "depends_on": ["..."] }
```

Preserve existing entries exactly — only append. Validate the JSON after writing.

```bash
# Validate JSON
python3 -c "import json; json.load(open('tasks/manifest.json'))" || echo "!! Invalid JSON"
```

### 6. Verify consistency

After creating the task file and updating the manifest:

- Confirm `tasks/NNN.md` exists and follows the template.
- Confirm `manifest.json` has the new entry and is valid JSON.
- Confirm every file listed in "Files to Read" either already exists or will be created by a dependency.
- Confirm the task count in the manifest matches the number of `.md` task files in `tasks/`.

### 7. Report to user

Tell the user:

- The task number and title.
- What it depends on and why.
- A one-line summary of what the agent will do when it executes this task.

## Rules

- **Never modify existing task files.** You are only adding new ones.
- **Never reorder or renumber existing tasks.** New tasks go at the end.
- **Manifest updates:** You may add new entries AND update the `depends_on` array of existing entries to reference the new task. Never remove entries or change existing dependency relationships — only add new dependencies pointing to the new task.
- **When in doubt about dependencies, ask.** Wrong dependencies cause execution failures or merge conflicts.
- **When in doubt about scope, ask.** If the description is vague or could mean multiple things, clarify before creating the task.
