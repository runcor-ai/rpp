# Skill: Convert Specification to R++

> **Purpose**: Convert any natural language specification, requirements document, user story, or feature description into a valid R++ script.
> **When to use**: When the user provides a spec, brief, or description and wants it converted to R++ format.
> **Input**: Any specification in natural language, markdown, bullet points, tables, or mixed format.
> **Output**: A complete, valid R++ script following the v0.5 language reference.

---

## Conversion Process

Follow these steps in order. Do not skip steps.

### Step 1: Identify the output type and domain

Read the input spec and determine:
- **What is being built?** (app, API, pipeline, CLI tool, document, dashboard, etc.)
- **What language/framework?** (React, Python, Rust, Node.js, etc.)
- **Which R++ profile applies?** (`ui`, `api`, `data`, `cli`, or none)

Write the `TARGET` block first.

### Step 2: Extract named values

Scan the spec for values that appear more than once or represent configurable constants:
- Colours, sizes, spacing, font sizes → TOKENS
- Limits, thresholds, timeouts, retry counts → TOKENS
- Roles, statuses, categories → TOKENS
- Format strings, date formats → FORMAT
- Status-to-label mappings, code-to-message mappings → MAP

**Rule**: If a value appears twice, it belongs in TOKENS. NEVER leave raw literals that could be tokenised.

### Step 3: Extract data

Look for:
- Tables, lists, or enumerations of items (users, endpoints, fields, stages, commands)
- Mark as `DATA EXACT` if the spec says these are the specific items (not examples)
- Mark as `DATA` (no EXACT) if the items are illustrative

### Step 4: Define state

From the spec, extract:
- Initial values (defaults, starting states, empty selections)
- Computed values (totals, averages, counts, filtered sets)
- Sort orders and filter logic

Write the `INIT` block.

### Step 5: Define structure

Map the spec's organisational hierarchy into a `STRUCTURE` tree:
- What contains what?
- What repeats?
- What is optional or conditional?
- What runs in sequence?

Use composition patterns: `sequence`, `group N`, `contains`, `split`, `repeat`, `optional`, `switch`.

### Step 6: Define components

For each named unit in the STRUCTURE, write a `COMPONENT` block:
- Bind data sources with `SOURCE=`
- Specify properties, content, and rules
- Define `EMPTY` states
- Use enforcement keywords (`MUST`, `MUST NOT`, `ALWAYS`, `NEVER`, `ONLY`, etc.)

### Step 7: Define behavior

Extract logic, interactions, and rules:
- Event handlers (`on EVENT → ACTION`)
- State machines (`STATE { transitions }`)
- Flow sequences (`FLOW { step → step }`)
- Constraints (`CONSTRAINT: rule`)
- Use enforcement keywords for precision

### Step 8: Write the checklist

Convert every requirement, acceptance criterion, and constraint into a boolean assertion:
- Each item MUST be specific and falsifiable
- Use enforcement keywords (`MUST`, `MUST NOT`, `ALWAYS`, `NEVER`, `ONLY`, `EXACTLY`, `AT LEAST`, `AT MOST`)
- NEVER write vague assertions like "works correctly" or "handles errors"

### Step 9: Review

Before outputting, verify:
- [ ] Every value that appears more than once is in TOKENS
- [ ] Every COMPONENT referenced in STRUCTURE is defined
- [ ] Every variable in BEHAVIOR is declared in INIT or COMPONENT
- [ ] CHECKLIST items are binary (yes/no answer by inspection)
- [ ] Enforcement keywords are used in CHECKLIST, BEHAVIOR, and COMPONENT (not in TOKENS, FORMAT, MAP, DATA, STRUCTURE)
- [ ] DATA EXACT is used for specific items, DATA for illustrative items
- [ ] No blocks are empty

---

## Conversion Rules

### What goes where

| Spec content | R++ block |
|-------------|-----------|
| "Build a..." / "Create a..." / project description | `TARGET` |
| Colours, sizes, limits, thresholds, roles | `TOKENS` |
| Date formatting, string transforms, display functions | `FORMAT` |
| Status labels, code mappings, category lookups | `MAP` |
| Tables of items, endpoint lists, field definitions | `DATA` |
| Defaults, initial state, computed aggregates | `INIT` |
| Page layout, section hierarchy, component tree | `STRUCTURE` |
| What each section/view/function contains | `COMPONENT` |
| Click handlers, validation rules, flows, state machines | `BEHAVIOR` |
| Acceptance criteria, requirements, constraints | `CHECKLIST` |

### Handling ambiguity

When the input spec is vague, do NOT guess. Instead:

1. **Missing values**: Use a descriptive TOKENS placeholder like `TBD_primaryColor:#000000` and add a CHECKLIST item: `[ ] MUST replace TBD_primaryColor with actual brand colour`
2. **Unclear logic**: Write the rule as you understand it in BEHAVIOR and add a CHECKLIST item: `[ ] MUST verify: [your interpretation of the rule]`
3. **Missing data**: Add an empty DATA block with column headers and a comment: `// EXACT data to be provided`
4. **Scope uncertainty**: Include the broader interpretation and add: `[ ] MUST confirm scope: does this include [uncertain feature]?`

### Enforcement keyword selection guide

| Spec says | Use |
|-----------|-----|
| "required", "mandatory", "needs to", "has to" | `MUST` |
| "cannot", "prohibited", "don't", "never allow" | `MUST NOT` |
| "every", "all", "each", "without exception" | `ALWAYS` |
| "none", "no", "zero", "prevent all" | `NEVER` |
| "just", "exclusively", "limited to", "restricted to" | `ONLY` |
| "except", "except when", "but if" | `UNLESS` |
| "first", "prior to", "preceding" | `BEFORE` |
| "then", "following", "subsequent" | `AFTER` |
| "precise", "exactly", "must be N" | `EXACTLY` |
| "minimum", "no fewer than", "at a minimum" | `AT LEAST` |
| "maximum", "no more than", "up to" | `AT MOST` |
| "ideally", "if possible", "nice to have" | `PREFER` |
| "otherwise", "if that fails", "backup plan" | `FALLBACK` |
| "replaces", "supersedes", "instead of" | `OVERRIDE` |

---

## Output Format

Always output a single fenced code block containing the complete R++ script. Use comments (`//`) to explain non-obvious decisions. Order blocks conventionally:

```
TARGET → TOKENS → FORMAT → MAP → DATA → INIT → STRUCTURE → COMPONENT → BEHAVIOR → CHECKLIST
```

After the code block, provide a brief summary:
- Number of components defined
- Number of checklist items
- Any assumptions made or ambiguities flagged
- Suggested next steps (e.g., "provide EXACT data for the endpoints table")

---

## Example Conversion

**Input spec:**
> "Build me a task management dashboard in React. It should show a list of tasks with title, status (todo/in-progress/done), and assignee. Users can filter by status and click a task to see details. Use a dark theme."

**Output:**

```
TARGET {
  output: "Task management dashboard with filterable task list and detail view"
  lang: React + TypeScript
  profile: ui
}

TOKENS {
  bgPrimary:#0d1117 | bgSecondary:#161b22 | bgCard:#1c2028
  accent:#58a6ff | textPrimary:#e6edf3 | textMuted:#8b949e
  border:#30363d
  gapSm:8px | gapMd:16px | gapLg:24px
  fontSm:13px | fontBase:15px | fontLg:20px | fontXl:28px
  radiusSm:6px | radiusMd:8px
}

FORMAT {
  fmtDate(d) => d.toLocaleDateString("en-US", {month:"short", day:"numeric"})
}

MAP {
  STATUS_COLOR { todo:TOKENS.textMuted | in-progress:TOKENS.accent | done:#22c55e }
  STATUS_LABEL { todo:"To Do" | in-progress:"In Progress" | done:"Done" }
}

TASKS EXACT:
// EXACT data to be provided — placeholder rows below
| id | title           | status      | assignee     |
|----|-----------------|-------------|--------------|
| 1  | Design homepage | in-progress | Alice        |
| 2  | Write API docs  | todo        | Bob          |
| 3  | Fix login bug   | done        | Alice        |

INIT {
  filterStatus = "all"
  selectedTask = null
  filtered     = tasks filtered by filterStatus
  taskCount    = count(tasks)
}

STRUCTURE {
  App: sequence
    Header
    MainContent: split
      [TaskList] [TaskDetail: optional]
}

COMPONENT Header {
  "Tasks" fontSize=fontXl bold color=textPrimary
  "{taskCount} tasks" fontSize=fontSm color=textMuted
}

COMPONENT TaskList SOURCE=filtered {
  <select> filterStatus: all | todo | in-progress | done
  columns: Title | Status | Assignee
  Status cell: pill badge | bg=STATUS_COLOR[status] | label=STATUS_LABEL[status]
  selected row: bg=bgCard | borderLeft=2px solid accent
  EMPTY: "No tasks match the current filter"
}

COMPONENT TaskDetail SOURCE=selectedTask {
  title: fontSize=fontLg bold
  status: pill badge | bg=STATUS_COLOR[status]
  assignee: textMuted
  EMPTY: "Select a task to view details"
}

BEHAVIOR {
  on click(task) → set selectedTask = task
  on click(task) when selectedTask.id === task.id → set selectedTask = null
  on change(filterStatus) → set selectedTask = null
  MUST update filtered list AFTER filterStatus changes
  MUST NOT lose selected state when filter doesn't affect selected task
}

CHECKLIST {
  [ ] MUST use all TOKENS names (NEVER raw colour or size literals)
  [ ] MUST include EXACTLY 3 EXACT task rows with correct values
  [ ] MUST treat "all" as passthrough in filter
  [ ] MUST show STATUS_LABEL text (NEVER raw status strings)
  [ ] MUST use STATUS_COLOR for badge backgrounds
  [ ] MUST NOT hardcode any computed values
  [ ] MUST show "Select a task to view details" when no task selected
  [ ] MUST show "No tasks match the current filter" when filtered list is empty
  [ ] ALWAYS use dark theme colours from TOKENS (NEVER light backgrounds)
}
```

**Summary**: 4 components, 9 checklist items. Placeholder EXACT data provided — replace with real tasks. Dark theme tokens extracted from spec requirement.
