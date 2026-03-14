# R++

**A structured specification language for LLMs.**

[Try the Playground](https://rpp.runcor.ai) · [Website](https://runcor.ai/rpp)

R++ eliminates ambiguity from LLM prompts. Instead of writing prose that an LLM has to interpret, you write a structured script with typed blocks, named values, and explicit assertions. The LLM reads the script and produces exactly what it describes.

## The Problem

Natural language prompts are ambiguous. An LLM reading "build me a dashboard with a user table" has to guess at colours, layout, data shape, interactions, error states, and dozens of other decisions. Each guess is a chance to get it wrong.

## The Solution

R++ gives every decision a name and a place:

```
TARGET {
  output: "Admin dashboard with user management"
  lang: React + TypeScript
  profile: ui
}

TOKENS {
  primary:#0d1117 | accent:#58a6ff
  gapMd:16px | fontBase:15px
}

COMPONENT UserTable SOURCE=users {
  columns: Name | Role | Status
  Status cell: pill badge | bg=STATUS_COLOR[status]
  EMPTY: "No users match the current filter"
}

CHECKLIST {
  [ ] MUST use all TOKENS names (NEVER raw literals)
  [ ] MUST include EXACTLY 5 data rows with correct values
  [ ] MUST NOT hardcode computed values
}
```

No guessing. Every value is named. Every assertion is explicit. Every requirement is verifiable.

## How It Works

You write an R++ script. You hand it to an LLM (Claude, GPT, Gemini, etc.) as a prompt. The LLM reads the structured blocks and produces the output they describe — a React component, a REST API, a data pipeline, a CLI tool, or any other artifact.

R++ is domain-agnostic. The same block structure works for any output type. Domain-specific vocabulary is provided through optional **profiles** (`ui`, `api`, `data`, `cli`), not baked into the core language.

## Block Types

| Block | Purpose |
|-------|---------|
| `TARGET` | Declare what is being built and which profile applies |
| `TOKENS` | Define named constants used throughout |
| `FORMAT` | Define formatting/transformation functions |
| `MAP` | Define lookup tables |
| `DATA` | Provide input data (use `EXACT` for verbatim reproduction) |
| `INIT` | Declare state and derived values |
| `STRUCTURE` | Describe how the output is composed |
| `COMPONENT` | Describe individual units of output |
| `BEHAVIOR` | Describe logic, interactions, and flows |
| `CHECKLIST` | Assert what must be true in the output |

## Enforcement Keywords

R++ includes precise enforcement keywords that eliminate vague instructions:

| Keyword | Meaning |
|---------|---------|
| `MUST` | Required. Non-negotiable. |
| `MUST NOT` | Prohibited. Hard block. |
| `ALWAYS` | Every instance, no exceptions. |
| `NEVER` | Zero instances, no exceptions. |
| `ONLY` | Exactly this, nothing else. |
| `EXACTLY` | Precise count or value. |
| `AT LEAST` | Lower bound. |
| `AT MOST` | Upper bound. |
| `UNLESS` | Exception to a rule. |
| `BEFORE` / `AFTER` | Ordering constraints. |
| `PREFER` | Soft preference. |
| `FALLBACK` | Recovery path when primary fails. |

No "should", "try to", or "ideally". Every keyword has a binary yes/no answer.

## Profiles

| Profile | Domain | Adds |
|---------|--------|------|
| `ui` | Web apps, dashboards, mobile | Style shorthands, layout patterns, chart/table/input syntax |
| `api` | REST APIs, GraphQL, RPC | Endpoint spec, request/response shapes, validation rules |
| `data` | ETL, pipelines, migrations | Source/dest config, transform rules, quality checks |
| `cli` | Command-line tools | Flag/argument spec, subcommands, output formatting |

## Files

| File | Description |
|------|-------------|
| [`rpp-language-reference-v0.5.md`](rpp-language-reference-v0.5.md) | Complete language reference with syntax, examples, and full templates |
| [`skill-spec-to-rpp.md`](skill-spec-to-rpp.md) | Claude Code skill that converts any natural language spec into R++ |

## Using the Claude Skill

Drop `skill-spec-to-rpp.md` and `rpp-language-reference-v0.5.md` into your project. When you give Claude a spec or feature description, it will convert it into a valid R++ script following the 9-step conversion process.

## Quick Example

**Input** (natural language):
> "Build a REST API for order management with CRUD endpoints, admin-only delete, and pagination."

**Output** (R++):
```
TARGET {
  output: "Order management REST API with CRUD and pagination"
  lang: Python + FastAPI
  profile: api
}

TOKENS {
  pageSize:20 | maxPageSize:100
  roleAdmin:"admin" | roleUser:"user"
  apiBase:"/api/v1"
}

ENDPOINTS EXACT:
| method | path             | handler       | auth  |
|--------|------------------|---------------|-------|
| GET    | /orders          | list_orders   | user  |
| POST   | /orders          | create_order  | user  |
| GET    | /orders/:id      | get_order     | user  |
| PUT    | /orders/:id      | update_order  | user  |
| DELETE | /orders/:id      | delete_order  | admin |

BEHAVIOR {
  MUST validate input BEFORE executing handler
  MUST NOT allow DELETE UNLESS role === TOKENS.roleAdmin
  CONSTRAINT: page MUST be AT LEAST 1
  CONSTRAINT: limit MUST be AT MOST TOKENS.maxPageSize
}

CHECKLIST {
  [ ] MUST implement EXACTLY 5 endpoints with correct methods
  [ ] MUST ALWAYS check auth BEFORE handler
  [ ] DELETE MUST be restricted to ONLY TOKENS.roleAdmin
  [ ] MUST NOT expose internal errors in responses
  [ ] MUST use TOKENS for all limits and roles (NEVER raw literals)
}
```

## License

MIT — see [LICENSE](LICENSE) for details.

---

<p align="center">
  <a href="https://rpp.runcor.ai">Playground</a> &middot;
  <a href="https://runcor.ai/rpp">Website</a> &middot;
  <a href="https://github.com/runcor-ai/rpp">GitHub</a>
</p>
