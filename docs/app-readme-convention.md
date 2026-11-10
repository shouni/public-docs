# Application README Convention

Decides how the READMEs of applications running on Cloud Run as web / worker services are structured.
It is for people starting a new application and people fixing an existing application's README.
After reading it you can decide mechanically what goes in the README, what is sent to CLAUDE.md and
what is not written at all, and judge for yourself where an existing README departs from the
convention.

Library READMEs are covered by [library-readme-convention.md](library-readme-convention.md). It is a
deliberately different convention that does not fix the skeleton (why: its 3.1). The languages
(English `README.md`, Japanese `README.ja.md`, English comments, messages and test names) are set there
in [2.7](library-readme-convention.md#27-languages) and apply here unchanged.

---

## 1. Why align them

When the five applications were lined up on 2026-09-04, their READMEs ranged from 268 to 580 lines and
agreed on neither the order nor the wording of headings. The real difference was not the headings but
**what the README was for**.

- The newer apps wrote usage in the README and reasons in CLAUDE.md (README 268 lines / CLAUDE.md 477).
- The older apps wrote design reasons into the README and had a thin CLAUDE.md (README 580 lines /
  CLAUDE.md 121). Readers looking for how to run it waded through design essays before reaching the
  environment variables.
- Technology-stack tables were copies of `go.mod` and stayed behind when a library was swapped.
- A renamed type's old name was left in one README in 10 places. Nobody rereads reason essays, so
  nobody notices when they rot.

The README's reader wants to run it; the CLAUDE.md reader wants to change it. One document for two
readers is too long for both.

---

## 2. Convention

### 2.1 Division of roles

| Document | Reader | What it says | Size |
|---|---|---|---|
| README.md (+ README.ja.md) | Someone who wants to run it | What it does, how to run it, how to call it | 250-350 lines |
| CLAUDE.md | Someone who wants to change it | Why it is so, rejected alternatives, holes stepped in, places not to touch | No limit |
| public-docs | Someone reading across apps | Conventions spanning apps, and their reasons | — |

There is one test: **a sentence containing "why", "used to", "rejected" or "it actually happened that"
belongs in CLAUDE.md.** The README keeps only the facts users have to follow as a result of the
decision (e.g. "the script is not put in the payload; only `job_id` is passed" is README; "we hit the
1 MB limit" is CLAUDE.md).

### 2.2 Skeleton

The wording, order and nesting of headings are fixed. If you want to add a section, first doubt the
test in 2.1.

```markdown
# <emoji> <app name>
<badges>

English | [日本語](README.ja.md)

## 🚀 About - <one-line claim>
## 📦 Usage
### 1. Configuration
### 2. Running
### 3. HTTP endpoints
### 4. Task payload
## 🔄 Sequence diagram
## 🌳 Project structure
## 📜 License
```

### 2.3 What each section holds

| Section | What it says | Form |
|---|---|---|
| About | What it makes from what input; what is distinctive (only what shows in users' operations); the web / worker split | Two or three paragraphs. At most five bullets if features are listed |
| 1. Configuration | Environment variables, **in four tables by role** (required for every role / required for web / required for worker / optional). Defaults and ordering constraints between values (the timeout ladder and so on) go here | `Variable \| Description` tables |
| 2. Running | The start command, and what each `SERVER_ROLE` assembles and which routes it exposes | Table |
| 3. HTTP endpoints | Every route. What sits outside authentication, representation by `Accept`, whether CSRF applies | `Method \| Path \| Purpose` table |
| 4. Task payload | The role and required fields of each `command`, a JSON example | Table + code block |
| Sequence diagram | The actual order from submission to notification, detailed enough to see that queued is recorded before enqueueing | Mermaid `sequenceDiagram` |
| Project structure | The directories under `internal/` with a one-line role | A `text` block written from `ls` |
| License | The license and attribution of materials used | Bullets |

### 2.4 What not to write

| Do not write | Reason | Instead |
|---|---|---|
| A technology-stack table | A copy of `go.mod`; stays behind when something is swapped | `go.mod` and [libraries.md](libraries.md) |
| "Principles" or "design philosophy" sections | Tend to read like marketing and tie to no user operation | Facts into About, reasons into CLAUDE.md |
| Architecture explanations | Content for the person who wants to change it | CLAUDE.md |
| Hand-copied lists (other than environment variables, routes and commands) | They drift from the real thing | How to find them (`grep`) |
| Copies of the cross-app conventions (URL naming, workers) | Left behind when the convention changes. "No `/api/` prefix" and "record queued before enqueueing" are the conventions' business | One line linking the convention. The README holds the app's own route table |
| Private repository names in a public repository | Puts private names in a public document | Role names (the music service, the MCP gateway) |
| Relative dates | See [documentation-guideline.md](documentation-guideline.md#7-leave-a-trail-of-changes) | Absolute dates (`2026-09-04`) |

The environment variable, route and command tables are hand-written lists too, but they are the point
of the README, so they are the exception. That is why they go through the check in 2.5.

### 2.5 Checking the facts

Each time the README's tables are written, they are checked against the code.

| Table | Checked against |
|---|---|
| Environment variables | `internal/config` (`env:"..."` tags) |
| Routes | `internal/server/router.go` |
| `command` | The command constants in `internal/domain` |
| Project structure | `ls internal` |
| Type and function names | `grep` that they exist. When renaming, fix the README (both languages) in the same commit |

---

## 3. Rejected alternatives

### 3.1 Keeping design reasons in the README

```text
Adopted: move reasons to CLAUDE.md
Reason: the readers differ. People who want to run it skip the reasons, and people
        who want to change it do not read the README. Writing them in both means
        maintaining two copies, and only one gets fixed
Revisit when: never
```

### 3.2 Aligning only the headings and leaving the content

```text
Adopted: sort the content by the test in 2.1 too
Reason: a README whose roles are reversed (reasons in the body, a thin CLAUDE.md)
        stays long with aligned headings. Same headings over content of a different
        nature confuses readers more
Revisit when: never
```

### 3.3 Keeping the technology-stack table

```text
Adopted: delete it
Reason: a copy of go.mod and libraries.md. When a library is swapped it is left
        behind as a third place
Revisit when: never
```

---

## 4. Finding READMEs that depart from the convention

```bash
# Headings that differ from the skeleton
grep -n '^## \|^### ' README.md
# Reasons left in the README
# Note: ordinary phrases such as "the string actually sent" match too; read the content, not the count
grep -n -i 'why\|used to\|rejected\|actually' README.md
# A technology-stack table left over
grep -n -i 'tech stack\|technology stack' README.md
# Old names left after a rename (example: step types that were renamed to Step)
grep -n 'Filter\b' README.md
```

The language checks (the switch, Japanese left in `README.md`) are in
[library-readme-convention.md §4](library-readme-convention.md#4-finding-readmes-that-depart-from-the-convention).

---

## Changelog

- 2026-09-04: First version. The READMEs of the five applications were compared, and the skeleton of
  the newest one (usage only, reasons in CLAUDE.md) became the convention.
- 2026-11-10: Translated to English. The skeleton's headings are now the English ones with a language
  switch; languages follow library-readme-convention.md 2.7.
