# R++ Scripting Language
# Language Reference — v0.5

---

## What R++ Is

R++ is a structured specification language for LLMs. You write an R++ script and hand it to an LLM as a prompt. The LLM reads the script and produces the output it describes.

R++ is domain-agnostic. The same block structure can specify a React component, a REST API, a CLI tool, a data pipeline, a document, a database schema, or any other artifact an LLM can generate. Domain-specific vocabulary is provided through optional **profiles**, not baked into the core language.

The core problem R++ solves: natural language prompts are ambiguous. An LLM reading prose has to guess at intent, fill in gaps, and make assumptions. R++ eliminates those gaps. Every block has a defined role. Every value is named. Every assertion is explicit. The LLM has less to invent.

---

## Structure

An R++ script is a sequence of named **blocks**. Each block has a keyword, an optional name, and a body delimited by `{ }`.

```
KEYWORD {
  ...
}

KEYWORD Name {
  ...
}
```

Blocks can appear in any order, but the conventional order is:

```
TARGET       →  declare what is being built and which profiles apply
TOKENS       →  define named constants used throughout
FORMAT       →  define how values should be rendered
MAP          →  define lookup tables
DATA         →  provide input data
INIT         →  declare state and derived values
STRUCTURE    →  describe how the output is composed
COMPONENT    →  describe individual units of output
BEHAVIOR     →  describe logic, interactions, and flows
CHECKLIST    →  assert what must be true in the output
```

Not every block is required. Use what the task needs.

---

## TARGET

Declares what the script produces and which domain profiles to activate. This is the first block the LLM reads and it frames everything that follows.

```
TARGET {
  output: description of what to generate
  lang: target language or format
  profile: profileName
}
```

`output` is a short description of the artifact being built. `lang` names the target language, format, or runtime. `profile` activates a domain-specific vocabulary extension (see Profiles).

Multiple profiles can be activated:

```
TARGET {
  output: "Single-page admin dashboard for user management"
  lang: React + TypeScript
  profile: ui
}
```

```
TARGET {
  output: "REST API for order management with CRUD endpoints"
  lang: Python + FastAPI
  profile: api
}
```

```
TARGET {
  output: "ETL pipeline that normalises vendor CSVs into warehouse schema"
  lang: Python
  profile: data
}
```

```
TARGET {
  output: "CLI tool for batch image resizing"
  lang: Rust
  profile: cli
}
```

If no profile is specified, only the core language is available. The LLM interprets COMPONENT bodies as plain specifications without domain-specific shorthands.

---

## TOKENS

Defines named constants. Any value that appears more than once in the output should be named here. The LLM must use these names — not the raw values — everywhere in the output.

```
TOKENS {
  name:value | name:value | name:value
  name:value
}
```

`|` separates tokens on the same line. Grouping by category on separate lines is conventional but not required.

TOKENS is domain-agnostic. What you name depends on what you're building:

```
// UI tokens
TOKENS {
  primary:#0d1117 | secondary:#161b22 | accent:#58a6ff
  gapSm:8px | gapMd:16px | gapLg:24px
  fontSm:13px | fontBase:15px | fontXl:32px
}
```

```
// API tokens
TOKENS {
  maxRetries:3 | timeoutMs:5000 | pageSize:20
  roleAdmin:"admin" | roleViewer:"viewer" | roleEditor:"editor"
  apiBase:"/api/v2"
  rateLimit:100 | ratePeriod:"1m"
}
```

```
// Data pipeline tokens
TOKENS {
  batchSize:1000 | maxParallel:4
  srcEncoding:"utf-8" | destEncoding:"utf-8"
  nullMarker:"N/A" | skipHeader:true
}
```

```
// CLI tokens
TOKENS {
  defaultFormat:"png" | maxWidth:4096 | maxHeight:4096
  quality:85 | threadCount:4
  verboseFlag:"--verbose" | quietFlag:"--quiet"
}
```

The LLM generates a named constant block from TOKENS and references names throughout, never the raw values.

---

## FORMAT

Defines named formatting or transformation functions. The LLM generates these as callable helpers and uses them wherever values need to be rendered or transformed.

```
FORMAT {
  functionName(param) => expression
}
```

```
FORMAT {
  fmtCalls(n)    => n.toLocaleString("en-US")
  fmtMs(n)       => n + "ms"
  fmtPct(n)      => n.toFixed(1) + "%"
  fmtDate(d)     => d.toLocaleDateString("en-US", {year:"numeric", month:"short", day:"numeric"})
  fmtUSD(n)      => "$" + n.toLocaleString("en-US", {minimumFractionDigits:2})
  truncate(s,n)  => s.length > n ? s.slice(0,n) + "…" : s
  slugify(s)     => s.toLowerCase().replace(/\s+/g, "-")
  sanitize(s)    => s.replace(/[<>&"]/g, "")
}
```

FORMAT expressions use the syntax of the target language declared in TARGET. If TARGET specifies Python, write Python expressions. If it specifies TypeScript, write TypeScript.

---

## MAP

Defines named lookup tables. Maps a set of keys to values. Values should reference TOKENS names where applicable.

```
MAP {
  MAP_NAME { key:value | key:value | key:value }
}
```

```
MAP {
  STATUS_LABEL   { active:"Active" | inactive:"Inactive" | pending:"Pending Review" }
  HTTP_LABEL     { 200:"OK" | 404:"Not Found" | 500:"Server Error" }
  PRIORITY_LEVEL { high:1 | medium:2 | low:3 }
  EXIT_CODE      { success:0 | warning:1 | error:2 }
}
```

Usage elsewhere: `MAP_NAME[key]`

---

## DATA

Provides input data as markdown tables. Two variants:

### DATA EXACT

The `EXACT` qualifier is a contract: every row, every field value, must appear in the output verbatim. The LLM must not invent, estimate, reorder, or omit any value.

```
DATA EXACT:
| field | field | field |
|-------|-------|-------|
| val   | val   | val   |
```

### Named data set

A label before the table names the dataset and optionally describes it.

```
LABEL (description):
| field | field |
|-------|-------|
| val   | val   |
```

```
ENDPOINTS EXACT:
| method | path            | handler        | auth     |
|--------|-----------------|----------------|----------|
| GET    | /users          | listUsers      | viewer   |
| POST   | /users          | createUser     | admin    |
| GET    | /users/:id      | getUser        | viewer   |
| PUT    | /users/:id      | updateUser     | editor   |
| DELETE | /users/:id      | deleteUser     | admin    |

TRANSFORMS (pipeline stages):
| stage     | input         | output        | rule                   |
|-----------|---------------|---------------|------------------------|
| normalize | raw_date      | iso_date      | parse any date to ISO  |
| validate  | email         | email         | reject invalid format  |
| enrich    | zip_code      | region        | lookup from postal DB  |
```

---

## INIT

Declares the operational state of the output: initial variable values, computed derivations, filter logic, and sort order. The LLM generates these as constants or initial state at the start of execution.

```
INIT {
  variable = value
  computed = expression    // optional comment
}
```

### Variables

Simple assignment declares the starting value:

```
selectedId    = null
currentPage   = 1
searchQuery   = ""
filterStatus  = "all"
```

### Computed expressions

| Syntax | Meaning |
|--------|---------|
| `sum(field)` | Sum of that field across all records |
| `mean(field)` | Arithmetic mean |
| `round(expr)` | Round to nearest integer |
| `min(field)` / `max(field)` | Extremes |
| `count where field === value` | Filter and count |
| `count where field >= n` | Numeric filter and count |
| `records filtered by var AND var` | Both conditions must match |
| `records filtered by var OR var` | Either condition matches |
| `set sorted by field DESC` | Descending sort (source not mutated) |
| `set sorted by field ASC` | Ascending sort |

`"all"` used as a filter value means: match everything (passthrough).

```
INIT {
  filterStatus   = "all"
  filterMethod   = "all"
  selectedId     = null
  totalRevenue   = sum(revenue)                       // 254500
  avgOrderValue  = round(mean(order_value))           // 312
  activeUsers    = count where status === "active"    // 847
  filtered       = users filtered by filterStatus AND filterRole
  sorted         = filtered sorted by last_login DESC
}
```

---

## STRUCTURE

Describes how the output is composed as an indented tree. Tells the LLM the organisational hierarchy before describing what each part contains.

STRUCTURE replaces the concept of "layout" with abstract **composition patterns** that work for any output type — not just spatial arrangement.

```
STRUCTURE {
  Root
    ComponentA
    ComponentB: pattern params
      [Part1] [Part2]
    ComponentC
}
```

### Composition patterns

| Pattern | Meaning |
|---------|---------|
| `sequence` | Ordered list of items, one after another |
| `group N` | N parallel items (columns, fields, slots — context-dependent) |
| `group "template"` | Explicit sizing/weight template |
| `contains` | Parent-child containment |
| `split` | Two-part division |
| `repeat` | Iterated over a data source |
| `optional` | Present only when a condition is met |
| `switch` | One-of-N based on a condition |

### Applying properties

Properties follow `key=value` or reference token names:

```
Root: padding=TOKENS.gapLg | sequence
```

`|` separates co-applied properties on the same line. It is **not** logical OR.

### Examples

**UI application:**
```
STRUCTURE {
  App: sequence
    Header
    KPIRow: group 4
      [TotalUsers] [ActiveSessions] [Revenue] [ErrorRate]
    MainContent: split
      [DataTable] [Sidebar]
    Footer
}
```

**REST API:**
```
STRUCTURE {
  API: sequence
    Middleware
    Routes: sequence
      [UserRoutes] [OrderRoutes] [ProductRoutes]
    ErrorHandler
}
```

**Data pipeline:**
```
STRUCTURE {
  Pipeline: sequence
    Ingest
    TransformChain: sequence
      [Normalize] [Validate] [Enrich] [Deduplicate]
    Load
    Report
}
```

**CLI tool:**
```
STRUCTURE {
  CLI: contains
    GlobalFlags
    Commands: switch
      [Resize] [Convert] [Compress] [Info]
}
```

**Document:**
```
STRUCTURE {
  Document: sequence
    FrontMatter
    Body: sequence
      [Introduction] [Analysis] [Recommendations]
    Appendix: optional
}
```

---

## COMPONENT

Describes what a named unit of the output contains. Each COMPONENT generates a discrete, self-contained piece of the output — a function, a class, a route handler, a document section, a UI view, a pipeline stage, or whatever the target format requires.

`COMPONENT` is the core keyword. `SECTION` and `VIEW` are aliases (all three are valid; use whichever reads best for the domain).

```
COMPONENT Name {
  ...spec lines...
}

COMPONENT Name SOURCE=dataVariable {
  ...spec lines...
}
```

`SOURCE=` binds an input dataset to the component.

### Spec lines

Inside a COMPONENT, lines describe what the output should contain. The syntax is intentionally flexible — the LLM interprets lines in the context of the TARGET.

**Property assignment:**
```
name: value
name: value | value | value    // co-applied properties
```

**Static text:**
```
"Literal text to include"
```

**Dynamic interpolation:**
```
"{variable} units processed"
"Showing {count} of {total} results"
```

**Conditional logic (use `;;` to separate branches):**
```
field >= threshold → result ;; field >= lower → result ;; else → result
```

The `;;` separator is unambiguous — it only appears in conditional chains and cannot be confused with the `|` property separator.

```
status_code: code >= 500 → "error" ;; code >= 400 → "client_error" ;; else → "ok"
log_level: severity === "critical" → TOKENS.error ;; severity === "warning" → TOKENS.warn ;; else → TOKENS.info
```

**Enumerated items:**
```
fields: name | email | role | created_at
accepts: "application/json" | "application/xml"
```

**EMPTY keyword** — defines what happens when a component has no data:
```
EMPTY: "No records found"
EMPTY: return empty array
EMPTY: skip stage and log warning
```

### Domain-specific spec lines

When a profile is active, additional spec-line syntaxes become available inside COMPONENT. See the Profiles section for details.

---

## BEHAVIOR

Describes logic, interactions, state transitions, and flows. BEHAVIOR separates *what happens* from *what things are* (COMPONENT).

```
BEHAVIOR {
  ...rules...
}

BEHAVIOR Name {
  ...rules...
}
```

### Event rules

```
on EVENT → ACTION
on EVENT → ACTION | ACTION    // multiple actions
on EVENT when CONDITION → ACTION
```

```
BEHAVIOR {
  on click(row) → set selectedId = row.id
  on click(row) when selectedId === row.id → set selectedId = null
  on change(searchQuery) → set currentPage = 1
  on submit(form) → call createUser(form.data) | set formOpen = false
}
```

### State transitions

```
STATE stateName {
  initial: value
  VALUE → VALUE when CONDITION
  VALUE → VALUE on EVENT
}
```

```
BEHAVIOR OrderLifecycle {
  STATE status {
    initial: "draft"
    "draft"      → "submitted"   on submit
    "submitted"  → "approved"    when approver !== null
    "submitted"  → "rejected"    on reject
    "approved"   → "fulfilled"   when all items_shipped
    ANY          → "cancelled"   on cancel when status !== "fulfilled"
  }
}
```

### Flow sequences

For multi-step processes:

```
FLOW name {
  step1 → step2 → step3
  step2 FAIL → errorHandler
}
```

```
BEHAVIOR IngestFlow {
  FLOW process {
    readSource → validate → transform → load → report
    validate FAIL → logError | skipRecord
    transform FAIL → quarantine
    load FAIL → retry(3) | abort
  }
}
```

### Constraints

Rules that must hold at runtime:

```
CONSTRAINT: expression
```

```
BEHAVIOR {
  CONSTRAINT: currentPage >= 1
  CONSTRAINT: pageSize <= TOKENS.maxPageSize
  CONSTRAINT: user.role IN [TOKENS.roleAdmin, TOKENS.roleEditor, TOKENS.roleViewer]
  CONSTRAINT: retryCount <= TOKENS.maxRetries
}
```

---

## CHECKLIST

A list of boolean assertions. The LLM verifies every item before finalising output. This is the enforcement mechanism — it converts the spec into machine-verifiable conditions.

```
CHECKLIST {
  [ ] assertion
  [ ] assertion
  ...
}
```

Each item must be **specific and falsifiable**. The LLM should be able to answer yes/no by inspecting the output directly.

```
CHECKLIST {
  [ ] all TOKENS names used throughout (no raw literals in output)
  [ ] all EXACT data rows present with correct values
  [ ] all computed values derived at run time (not hardcoded)
  [ ] filter uses AND logic; "all" matches any
  [ ] sort is immutable (source array not mutated)
  [ ] MAP values reference TOKENS names
  [ ] every COMPONENT referenced in STRUCTURE is defined
  [ ] every BEHAVIOR event references a declared variable or component
  [ ] EMPTY states render the specified fallback
  [ ] assertions verified before output finalised
}
```

### Wording precision

The wording of CHECKLIST items is load-bearing. Small differences produce different outputs:

| Vague — avoid | Precise — use |
|---------------|--------------|
| `[ ] TOKENS is first` | `[ ] TOKENS const is first declaration after imports` |
| `[ ] errors are handled` | `[ ] every endpoint returns 400 for invalid input and 500 for server errors` |
| `[ ] validation works` | `[ ] email field rejects strings without @ using regex validation` |
| `[ ] pipeline is resilient` | `[ ] transform FAIL routes to quarantine; load FAIL retries 3 times then aborts` |

---

## Comments

```
// Full line comment — appears anywhere inside a block
totalRevenue = sum(revenue)   // expected: 254500
```

---

## Operator Reference

| Category | Operators / Keywords |
|----------|---------------------|
| Comparison | `===` `!==` `>=` `<=` `>` `<` `IN` |
| Logic | `AND` `OR` |
| Aggregates | `sum()` `mean()` `round()` `count where` `min()` `max()` |
| Sort | `sorted by field DESC` / `sorted by field ASC` |
| Filter passthrough | `"all"` — matches any value in a filter expression |
| Conditional | `condition → value ;; condition → value ;; else → value` |
| Co-apply | `\|` — separates co-applied properties (not logical OR) |
| Assignment | `=` in INIT, BEHAVIOR, and property lines |
| Token def | `:` in TOKENS block |
| Function def | `=>` in FORMAT block |
| Event | `on EVENT → ACTION` in BEHAVIOR |
| Transition | `STATE → STATE` in BEHAVIOR |
| Flow | `step → step` in BEHAVIOR FLOW |
| Enforcement | `MUST` `MUST NOT` `ALWAYS` `NEVER` `ONLY` `UNLESS` `EXACTLY` `AT LEAST` `AT MOST` in CHECKLIST, BEHAVIOR, COMPONENT |
| Preference | `PREFER` — soft preference (non-blocking) |
| Recovery | `FALLBACK` — what to do when primary fails |
| Ordering | `BEFORE` `AFTER` — sequence constraints |
| Replacement | `OVERRIDE` — explicitly replaces a prior rule |

---

## Profiles

Profiles add domain-specific vocabulary to the core language. They introduce additional spec-line syntaxes for COMPONENT, additional composition patterns for STRUCTURE, and additional CHECKLIST conventions.

A profile is activated in the TARGET block. The core language is always available regardless of profile.

### Profile: `ui`

For visual outputs — web apps, dashboards, mobile screens, design specs.

**Additional COMPONENT syntax:**

Style shorthands (expand to their semantic equivalents using TOKENS):

| Shorthand | Meaning |
|-----------|---------|
| `bgBase` / `bgRaised` / `bgCard` | Background from TOKENS |
| `border` | 1px border using TOKENS.border |
| `text` / `textMuted` | Text colour from TOKENS |
| `bold` | fontWeight 700 |
| `uppercase` | Text transform uppercase |
| `centered` | Text align center |
| `monospace` | Monospace font |

Layout properties:
```
container: bgRaised | border | radiusMd | padding=gapMd
title: "Section Title" fontSize=fontLg fontWeight=700 color=text
```

Table specification:
```
columns: Col1 | Col2 | Col3
header: bgBase | textMuted | fontXs | uppercase
Col2 cell: pill badge | bg=MAP[field] | color=bgBase
selected row: bg=bgCard | borderLeft=2px solid accent (toggle on click)
```

Input controls:
```
<select> varName: option1 | option2 | option3
<input>  varName  placeholder="Search..."
<toggle> varName  label="Show archived"
```

Chart specification:
```
type: AreaChart | height=260
dataKey=value | stroke=TOKENS.accent | fill=TOKENS.accent
XAxis dataKey=label | YAxis value | Tooltip
```

Chart types: `AreaChart` `BarChart` `LineChart` `PieChart` `RadarChart` `ScatterChart`

**Additional STRUCTURE patterns:**
| Pattern | Meaning |
|---------|---------|
| `stack` | Vertically stacked (alias for `sequence` with vertical implication) |
| `stack gap=token` | Stacked with spacing |
| `grid N-col` | N equal columns (alias for `group N` with grid implication) |
| `grid "template"` | Explicit column template (e.g., `"1fr 320px"`) |
| `inline` | Horizontally arranged |

---

### Profile: `api`

For backend services — REST APIs, GraphQL schemas, RPC interfaces.

**Additional COMPONENT syntax:**

Endpoint specification:
```
method: GET | POST | PUT | DELETE
path: "/users/:id"
auth: TOKENS.roleAdmin
```

Request/response shape:
```
request {
  body: { name:string, email:string, role:string }
  query: { page:int, limit:int }
  headers: { Authorization:string }
}

response 200 {
  body: { id:int, name:string, created_at:datetime }
}

response 400 {
  body: { error:string, details:array }
}
```

Validation rules:
```
validate: name required | email format=email | role IN MAP.VALID_ROLES
```

**Additional CHECKLIST conventions:**
```
[ ] every endpoint returns appropriate status codes (200, 400, 404, 500)
[ ] auth middleware applied to all protected routes
[ ] request validation runs before handler logic
[ ] error responses include structured error body
```

---

### Profile: `data`

For data pipelines — ETL, transforms, migrations, batch processing.

**Additional COMPONENT syntax:**

Source/destination:
```
source: type=csv | path=TOKENS.inputPath | encoding=TOKENS.srcEncoding
dest: type=postgres | table="users" | schema="public"
```

Transform rules:
```
transform {
  rename: old_field → new_field
  cast: date_string → date using FORMAT.parseDate
  derive: full_name = first_name + " " + last_name
  drop: internal_id | temp_flag
  filter: status !== "deleted"
}
```

Quality checks:
```
quality {
  not_null: id | email | created_at
  unique: id | email
  range: age >= 0 AND age <= 150
  format: email matches /^.+@.+\..+$/
}
```

**Additional CHECKLIST conventions:**
```
[ ] all source fields mapped or explicitly dropped
[ ] null handling defined for every non-nullable destination field
[ ] failed records routed to quarantine with error reason
[ ] row counts logged before and after each stage
```

---

### Profile: `cli`

For command-line tools — argument parsing, subcommands, output formatting.

**Additional COMPONENT syntax:**

Flag/argument specification:
```
flag: --verbose | -v | type=bool | default=false | help="Enable verbose output"
flag: --output | -o | type=path | required | help="Output directory"
arg: FILES | type=path[] | positional | help="Input files to process"
```

Subcommand:
```
command: resize
  arg: FILE | positional | required
  flag: --width | -w | type=int | default=TOKENS.maxWidth
  flag: --quality | -q | type=int | default=TOKENS.quality | range=1..100
```

Output formatting:
```
output {
  success: "{count} files processed in {elapsed}s"
  error: "Error: {message}" → stderr
  verbose: "[{timestamp}] {stage}: {detail}" when TOKENS.verboseFlag
}
```

**Additional CHECKLIST conventions:**
```
[ ] --help produces usage text listing all flags and subcommands
[ ] invalid flags produce specific error message and exit code 2
[ ] exit code 0 on success, 1 on handled error, 2 on usage error
[ ] stderr used for errors, stdout for output
```

---

## Full Script Templates

### Template: UI Dashboard

```
TARGET {
  output: "Admin dashboard with user management table"
  lang: React + TypeScript
  profile: ui
}

TOKENS {
  primary:#0d1117 | secondary:#161b22 | accent:#58a6ff
  gapSm:8px | gapMd:16px | gapLg:24px
  fontSm:13px | fontBase:15px | fontXl:32px
}

FORMAT {
  fmtDate(d) => d.toLocaleDateString("en-US", {month:"short", day:"numeric"})
}

MAP {
  STATUS_COLOR { active:TOKENS.accent | inactive:TOKENS.secondary }
}

USERS EXACT:
| id | name         | role   | status   |
|----|--------------|--------|----------|
| 1  | Alice Nguyen | admin  | active   |
| 2  | Bob Okafor   | viewer | inactive |

INIT {
  filterStatus = "all"
  filtered     = users filtered by filterStatus
}

STRUCTURE {
  App: sequence
    Header
    UserTable: contains
      [Filters] [Table] [Pagination]
}

COMPONENT Header {
  "User Management" fontSize=fontXl bold
}

COMPONENT Table SOURCE=filtered {
  columns: Name | Role | Status
  Status cell: pill badge | bg=STATUS_COLOR[status]
  EMPTY: "No users match the current filters"
}

CHECKLIST {
  [ ] MUST use all TOKENS names throughout (NEVER raw literals)
  [ ] MUST include EXACTLY 2 EXACT data rows with correct values
  [ ] MUST treat "all" as passthrough in filter logic
  [ ] MUST NOT hardcode computed values
}
```

### Template: REST API

```
TARGET {
  output: "User management REST API with CRUD"
  lang: Python + FastAPI
  profile: api
}

TOKENS {
  pageSize:20 | maxPageSize:100
  roleAdmin:"admin" | roleEditor:"editor" | roleViewer:"viewer"
  apiBase:"/api/v1"
}

FORMAT {
  slugify(s) => s.lower().replace(" ", "-")
}

MAP {
  PERMISSION { GET:TOKENS.roleViewer | POST:TOKENS.roleAdmin | PUT:TOKENS.roleEditor | DELETE:TOKENS.roleAdmin }
}

ENDPOINTS EXACT:
| method | path            | handler        | auth   |
|--------|-----------------|----------------|--------|
| GET    | /users          | list_users     | viewer |
| POST   | /users          | create_user    | admin  |
| GET    | /users/:id      | get_user       | viewer |
| PUT    | /users/:id      | update_user    | editor |
| DELETE | /users/:id      | delete_user    | admin  |

INIT {
  currentUser   = null
  defaultPage   = 1
  defaultLimit  = TOKENS.pageSize
}

STRUCTURE {
  API: sequence
    AuthMiddleware
    Routes: sequence
      [ListUsers] [CreateUser] [GetUser] [UpdateUser] [DeleteUser]
    ErrorHandler
}

COMPONENT ListUsers {
  method: GET
  path: TOKENS.apiBase + "/users"
  auth: TOKENS.roleViewer
  request {
    query: { page:int, limit:int, status:string }
  }
  response 200 {
    body: { users:array, total:int, page:int }
  }
  validate: limit <= TOKENS.maxPageSize
}

COMPONENT ErrorHandler {
  response 400 { body: { error:string, details:array } }
  response 404 { body: { error:"Not found" } }
  response 500 { body: { error:"Internal server error" } }
}

BEHAVIOR {
  CONSTRAINT: page MUST be AT LEAST 1
  CONSTRAINT: limit MUST be AT LEAST 1 AND AT MOST TOKENS.maxPageSize
  on request when !authenticated → ALWAYS respond 401
  on request when !authorized → ALWAYS respond 403
  MUST validate input BEFORE executing handler
  MUST NOT process request UNLESS authenticated
}

CHECKLIST {
  [ ] MUST implement EXACTLY 5 EXACT endpoints with correct methods and paths
  [ ] MUST ALWAYS check auth middleware BEFORE every handler
  [ ] MUST default pagination to page=1 limit=20
  [ ] MUST return 400 with structured error body on invalid input
  [ ] DELETE MUST return 204 (MUST NOT include body)
  [ ] MUST use all TOKENS names (NEVER raw literals for roles, limits, or base path)
  [ ] MUST NOT expose internal errors in response bodies
}
```

### Template: Data Pipeline

```
TARGET {
  output: "CSV-to-warehouse ETL pipeline with validation and error handling"
  lang: Python
  profile: data
}

TOKENS {
  batchSize:1000 | maxParallel:4
  srcPath:"./incoming" | destTable:"clean_users"
  nullMarker:"N/A" | skipHeader:true
}

FORMAT {
  parseDate(s) => datetime.strptime(s, "%m/%d/%Y").isoformat()
  normalizeEmail(s) => s.strip().lower()
}

TRANSFORMS EXACT:
| stage     | input      | output     | rule                       |
|-----------|------------|------------|----------------------------|
| normalize | raw_date   | iso_date   | parse any date to ISO 8601 |
| normalize | email      | email      | lowercase and strip        |
| validate  | email      | email      | reject if no @ present     |
| validate  | age        | age        | reject if < 0 or > 150    |
| enrich    | zip_code   | region     | lookup from postal DB      |

INIT {
  processedCount = 0
  errorCount     = 0
  quarantined    = []
}

STRUCTURE {
  Pipeline: sequence
    Ingest
    TransformChain: sequence
      [Normalize] [Validate] [Enrich]
    Load
    Report
}

COMPONENT Ingest {
  source: type=csv | path=TOKENS.srcPath | encoding="utf-8"
  batch: size=TOKENS.batchSize
  EMPTY: log "No files found in source directory" | exit 0
}

COMPONENT Validate {
  quality {
    not_null: id | email
    unique: id
    format: email matches /^.+@.+\..+$/
    range: age >= 0 AND age <= 150
  }
  on fail → quarantine(record, reason) | increment errorCount
}

COMPONENT Load {
  dest: type=postgres | table=TOKENS.destTable
  mode: upsert on id
  on fail → retry(3) | abort
}

COMPONENT Report {
  "{processedCount} records loaded, {errorCount} quarantined"
}

BEHAVIOR IngestFlow {
  FLOW process {
    Ingest → Normalize → Validate → Enrich → Load → Report
    Validate FAIL → quarantine
    Load FAIL → retry(3) | abort
  }
}

CHECKLIST {
  [ ] MUST implement EXACTLY 5 EXACT transform rules in correct order
  [ ] MUST ALWAYS route failed validation to quarantine with reason string
  [ ] MUST use TOKENS.batchSize (NEVER hardcode batch size)
  [ ] MUST use upsert (MUST NOT use insert) to handle reruns
  [ ] MUST print EXACTLY the counts of processed and quarantined
  [ ] MUST exit 0 on success, 1 on abort (NEVER exit 0 on failure)
  [ ] MUST ALWAYS log row counts BEFORE and AFTER each stage
}
```

### Template: CLI Tool

```
TARGET {
  output: "CLI image resizer with subcommands"
  lang: Rust
  profile: cli
}

TOKENS {
  defaultFormat:"png" | maxWidth:4096 | maxHeight:4096
  quality:85 | threadCount:4
}

MAP {
  FORMAT_EXT { "png":"png" | "jpeg":"jpg" | "webp":"webp" }
}

STRUCTURE {
  CLI: contains
    GlobalFlags
    Commands: switch
      [Resize] [Convert] [Info]
}

COMPONENT GlobalFlags {
  flag: --verbose | -v | type=bool | default=false | help="Enable verbose output"
  flag: --threads | -t | type=int | default=TOKENS.threadCount | help="Worker threads"
}

COMPONENT Resize {
  command: resize
  arg: FILES | positional | required | type=path[]
  flag: --width | -w | type=int | default=TOKENS.maxWidth
  flag: --height | -h | type=int | default=TOKENS.maxHeight
  flag: --quality | -q | type=int | default=TOKENS.quality | range=1..100
  flag: --output | -o | type=path | required | help="Output directory"
  output {
    success: "Resized {count} images → {outputDir}"
    error: "Error processing {filename}: {message}" → stderr
  }
}

COMPONENT Info {
  command: info
  arg: FILE | positional | required | type=path
  output {
    "{filename}: {width}x{height} {format} ({filesize})"
  }
}

BEHAVIOR {
  on invalid_flag → ALWAYS print usage | exit 2
  on file_not_found → ALWAYS print "Error: {path} not found" → stderr | exit 1
  CONSTRAINT: quality MUST be AT LEAST 1 AND AT MOST 100
  CONSTRAINT: width MUST be AT LEAST 1 AND AT MOST TOKENS.maxWidth
  MUST NOT silently ignore invalid input
  NEVER continue processing AFTER fatal error
}

CHECKLIST {
  [ ] --help MUST list all subcommands and global flags
  [ ] MUST validate quality range 1-100 BEFORE processing
  [ ] MUST ALWAYS write errors to stderr, ONLY output to stdout
  [ ] MUST exit 0 on success, 1 on error, 2 on usage (NEVER exit 0 on failure)
  [ ] MUST use TOKENS.threadCount (NEVER hardcode thread count)
  [ ] MUST print specific error on missing required args (MUST NOT crash with generic trace)
}
```

---

## Enforcement Keywords

Enforcement keywords eliminate ambiguity in CHECKLIST, BEHAVIOR, and COMPONENT blocks. Each keyword has a precise, binary meaning — the LLM can evaluate every one as true or false against its output.

### Keyword Reference

| Keyword | Meaning | LLM reads it as |
|---------|---------|-----------------|
| `MUST` | Required. Non-negotiable. | "If I don't do this, the output is wrong." |
| `MUST NOT` | Prohibited. Hard block. | "If I do this, the output is wrong." |
| `ALWAYS` | Every instance, no exceptions. | "Do this in 100% of cases, never skip." |
| `NEVER` | Zero instances, no exceptions. | "Do this in 0% of cases, no matter what." |
| `ONLY` | Exactly this, nothing else. | "This is the complete set. Don't add to it." |
| `UNLESS` | Exception to a rule. | "Follow the rule except when this condition is true." |
| `BEFORE` | Ordering constraint. | "This happens first, then the other thing." |
| `AFTER` | Ordering constraint. | "This happens second, after the other thing." |
| `EXACTLY` | Precise count or value. | "This number/value, not more, not less." |
| `AT MOST` | Upper bound. | "This many or fewer." |
| `AT LEAST` | Lower bound. | "This many or more." |
| `PREFER` | Soft preference. Not a failure if skipped. | "Do this if possible, but don't fail if you can't." |
| `FALLBACK` | What to do when the primary fails. | "If the preferred approach doesn't work, do this instead." |
| `OVERRIDE` | Explicitly replaces a prior rule. | "Ignore the earlier instruction, this one wins." |

### Usage in CHECKLIST

Enforcement keywords replace vague assertions with precise, falsifiable statements:

```
// Vague — avoid
CHECKLIST {
  [ ] tokens are used
  [ ] data rows are present
  [ ] errors are handled
  [ ] sort works correctly
}

// Precise — use
CHECKLIST {
  [ ] MUST use all TOKENS names (NEVER raw literals in output)
  [ ] MUST include EXACTLY 5 EXACT data rows with correct values
  [ ] MUST NOT return 200 on validation failure
  [ ] MUST NOT mutate source arrays during sort
  [ ] ALWAYS return structured error body on 4xx/5xx
  [ ] NEVER expose internal stack traces in error responses
  [ ] AT LEAST 1 retry BEFORE aborting on load failure
  [ ] AT MOST 3 retries per failed operation
}
```

### Usage in COMPONENT

Enforcement keywords constrain what a component contains or produces:

```
COMPONENT Validate {
  quality {
    MUST NOT accept null: id | email
    MUST be unique: id
    NEVER accept age < 0 OR age > 150
  }
  on fail → ALWAYS quarantine(record, reason)
  FALLBACK: log warning and skip record
}

COMPONENT DeleteUser {
  method: DELETE
  auth: ONLY TOKENS.roleAdmin
  response 204: MUST NOT include body
  BEFORE handler: ALWAYS check auth middleware
}
```

### Usage in BEHAVIOR

Enforcement keywords add precision to rules, constraints, and flows:

```
BEHAVIOR {
  MUST validate input BEFORE executing handler
  MUST NOT process request AFTER shutdown signal
  ALWAYS log request duration AFTER handler completes
  NEVER retry on 4xx errors (ONLY retry on 5xx)

  CONSTRAINT: page MUST be AT LEAST 1
  CONSTRAINT: limit MUST be AT MOST TOKENS.maxPageSize
  CONSTRAINT: retryCount MUST NEVER exceed TOKENS.maxRetries

  FLOW process {
    Ingest → Validate → Transform → Load → Report
    Validate FAIL → ALWAYS quarantine UNLESS errorCount > TOKENS.maxErrors
    Load FAIL → retry(3) | FALLBACK abort with error report
  }
}
```

### Where NOT to use enforcement keywords

Do not use enforcement keywords in blocks that are already inherently exact:

- **TOKENS** — values are definitions, not instructions. `primary:#0d1117` is already unambiguous.
- **FORMAT** — function definitions are exact by nature.
- **MAP** — lookup tables are exact by nature.
- **DATA EXACT** — the `EXACT` qualifier already enforces verbatim reproduction.
- **STRUCTURE** — composition is structural, not conditional.

### Combining keywords

Keywords can be combined for compound constraints:

```
MUST ALWAYS validate BEFORE processing
MUST NEVER return null UNLESS input is empty
ONLY retry on 5xx errors, NEVER on 4xx
AT LEAST 1 and AT MOST 3 retries BEFORE abort
```

The LLM evaluates compound constraints left to right. Each keyword narrows the instruction further.

---

## Core Design Principles

**1. Name everything once.**
Any value that appears more than once belongs in TOKENS. The output references names, never raw values. This makes changes global: update one token, the whole output changes.

**2. Blocks have single responsibilities.**
TARGET declares intent. TOKENS defines values. FORMAT defines transforms. DATA provides inputs. INIT declares state. STRUCTURE describes composition. COMPONENT describes content. BEHAVIOR describes logic. CHECKLIST verifies. Mixing these concerns creates ambiguity.

**3. CHECKLIST is the contract.**
A spec written in prose relies on the LLM interpreting intent. A CHECKLIST item is a boolean the LLM can verify by inspection. The CHECKLIST converts intent into assertions, and assertions are what get checked.

**4. EXACT means exact.**
When `EXACT` appears on a DATA block, it is a strict instruction: every row, every value, verbatim. Without it, the LLM may summarise, paraphrase, or drop records it considers redundant.

**5. COMPONENT blocks imply discrete units.**
Each COMPONENT becomes a named, self-contained unit in the output. This is structurally different from describing the same content in prose, which tends to produce a monolithic result.

**6. Core is domain-agnostic; profiles add domain vocabulary.**
The core language (TOKENS, FORMAT, MAP, DATA, INIT, STRUCTURE, COMPONENT, BEHAVIOR, CHECKLIST) works for any output type. Profiles like `ui`, `api`, `data`, and `cli` add the shorthands and conventions specific to a domain without polluting the core.

**7. `|` is co-apply; `;;` is branch.**
Inside property lines, `|` means "also apply this". In conditional expressions, `;;` separates branches. Logical OR in filter expressions is written `OR`. These three concepts never share a symbol.

**8. Enforcement keywords are binary.**
`MUST`, `MUST NOT`, `ALWAYS`, `NEVER`, `ONLY`, `EXACTLY` — each one has a yes/no answer. The LLM can verify every keyword against its output with no interpretation required. Vague words like "should", "try to", "ideally", or "consider" are not part of R++.

---

## Migration from v0.4

| v0.4 | v0.5 | Notes |
|------|------|-------|
| Plain CHECKLIST assertions | Enforcement keywords in CHECKLIST | `MUST`, `MUST NOT`, `ALWAYS`, `NEVER`, `ONLY`, `EXACTLY`, etc. |
| Plain CONSTRAINT syntax | Keyword-enhanced CONSTRAINT | `CONSTRAINT: page >= 1` → `CONSTRAINT: page MUST be AT LEAST 1` |
| Implicit enforcement | Explicit enforcement | Keywords make the LLM's obligations binary and verifiable |
| `PREFER` (new) | — | Soft preference replaces ambiguous "should" |
| `FALLBACK` (new) | — | Explicit failure recovery path |
| `OVERRIDE` (new) | — | Explicit rule replacement |

---

## Migration from v0.3

| v0.3 | v0.4 | Notes |
|------|------|-------|
| (implicit) | `TARGET` | New block — declare output type and profile |
| `LAYOUT` | `STRUCTURE` | Renamed; layout types replaced by abstract composition patterns |
| `SECTION` / `VIEW` | `COMPONENT` / `SECTION` / `VIEW` | `COMPONENT` is now the primary keyword; `SECTION` and `VIEW` remain as aliases |
| (implicit) | `BEHAVIOR` | New block — logic, events, flows, state machines |
| `cond → val \| cond → val` | `cond → val ;; cond → val` | Branch separator changed from `\|` to `;;` |
| Style shorthands in core | Style shorthands in `ui` profile | `bgBase`, `bold`, etc. now require `profile: ui` |
| `grid 3-col` | `group 3` (core) or `grid 3-col` (ui profile) | Grid/stack are now ui-profile aliases for core patterns |
