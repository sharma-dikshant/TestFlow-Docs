---
name: discover-project-context
description: >
  Use this skill whenever you need to gather context about a documented project
  before doing design, refactoring, or implementation work — especially before
  generating HLDs, LLDs, or onboarding-style summaries. Triggers: any task that
  benefits from knowing the project's existing architecture, ERD/data model, API
  conventions, data-access patterns, or prior design docs. Use it proactively
  whenever the user asks you to "design", "spec", "plan", or "build" something
  inside a project that has a `docs/` folder or an `mkdocs.yml` — the discovery
  step prevents you from inventing entities, contradicting the existing
  architecture, or breaking API conventions. Also use it when the user references
  "the project", "our codebase", or "the existing system" without giving context.
---

# Discover Project Context

A lightweight workflow for finding and synthesizing project documentation before
doing design or implementation work. Uses only native tools (`bash`, `view`) —
no scripts to run.

The output is an **internal context bundle** you hold in mind, not something you
dump to the user. Surface only what the user needs to confirm.

---

## Step 1 — Locate the project root

The project root is the directory containing either `mkdocs.yml` or a `docs/`
folder. Walk upward from the current working directory to find it.

```bash
# From any directory inside the project:
dir="$(pwd)"
while [ "$dir" != "/" ]; do
  if [ -f "$dir/mkdocs.yml" ] || [ -d "$dir/docs" ]; then
    echo "$dir"; break
  fi
  dir="$(dirname "$dir")"
done
```

If nothing is found, ask the user for the project root path before continuing.

---

## Step 2 — Enumerate documentation files

List every markdown file under the project root, including top-level files
like `hld-implementation-plan.md`.

```bash
cd <project_root>
find docs -name '*.md' -type f 2>/dev/null
find . -maxdepth 1 -name '*.md' -type f
```

---

## Step 3 — Categorize each file

Bucket the files by purpose using filename heuristics. Apply rules **in order**
— first match wins:

| Bucket                | Filename contains                         |
| --------------------- | ----------------------------------------- |
| `implementation_plan` | `implementation-plan`, `roadmap`          |
| `existing_hlds`       | `hld-design/`, `/hld/`, `hld-`            |
| `data_model`          | `erd`, `data-model`, `schema`, `entities` |
| `data_access`         | `data-access`, `repository`, `dao`        |
| `api_design`          | `api-design`, `/api/`, `endpoints`        |
| `architecture`        | `architecture`, `/arch`, `system-design`  |
| `overview`            | `overview`, `index`, `readme`             |
| `other`               | anything else                             |

Skip the `claude-agent-arch/` folder (and similar tooling subdirs) unless they
look directly relevant to the task at hand.

---

## Step 4 — Extract markdown outlines for foundational docs

For each file in `architecture`, `data_model`, `data_access`, `api_design`, and
`existing_hlds`, pull the headers to build a quick table of contents. This lets
you jump to relevant sections instead of reading full files.

```bash
# For each foundational doc:
grep -nE '^#{1,4} ' <file>
```

The output gives you `line:level text` pairs — enough to know what's in the
file before reading it.

---

## Step 5 — Grep for feature / topic mentions

If you're researching a specific feature or topic, find every place it's
already mentioned across the docs. Often you'll find a partial spec, a passing
constraint, or a related design decision worth respecting.

```bash
grep -rin "<feature or topic>" docs/ *.md 2>/dev/null
```

Open each hit at the cited line and read ±20 lines of surrounding context.

---

## Step 6 — Read the foundational docs

In this order:

1. **`docs/overview.md`** / **`docs/index.md`** — domain language, what the
   product does
2. **`docs/architecture.md`** — system layout the work must fit into
3. **`docs/erd.md`** (or whatever's in `data_model`) — **source of truth for
   entities. Do not redefine them.**
4. **`docs/data-access-pattern.md`** — how the project does DB access
5. **`docs/api-design.md`** and versioned variants — REST conventions, error
   envelope, auth header, versioning, pagination
6. **At least one file from `docs/hld-design/`** — for format and tone
   reference if you're producing a new design doc

Use the outlines from Step 4 to read selectively when files are large.

---

## Step 7 — Map task entities against the existing data model

After reading the ERD, classify any entity the task touches:

- **Reused entities** — already exist in the ERD. Cite by name with a link to
  the ERD section. **Never redefine.**
- **New entities** — needed but missing. Fully spec them in the output
  artifact and flag with `> **ERD Addition Required:**` so a human knows to
  update `docs/erd.md`.
- **Modified entities** — existing entities needing new fields. Flag with
  `> **ERD Modification Required:**` and list the additions.

---

## Step 8 — Synthesize the context bundle

Hold this internal summary before moving on to the actual task:

```
- Project root:     <absolute path>
- System fit:       where the work plugs into the architecture
- Reused entities:  [User, Test, ...] (with fields you'll touch)
- New entities:     [TestParticipant, ...] (to be spec'd in output)
- API conventions:  versioning style, error envelope, auth scheme
- Data access:      repository/DAO pattern, ORM, transaction style
- Format reference: which existing doc you'll structurally mirror
- Open gaps:        questions to ask the user before proceeding
```

Surface only the open gaps to the user. Carry the rest into the next phase
silently.

---

## Fallbacks

- **Project root not found:** ask the user for the path explicitly.
- **No `docs/` folder, just scattered markdown:** apply the same heuristics to
  `*.md` files anywhere under the root.
- **No existing HLDs / format references:** ask the user which file most
  closely matches the format they want for the output.
- **Feature has zero mentions in grep:** that's fine — it's new work. Note it
  and proceed.
