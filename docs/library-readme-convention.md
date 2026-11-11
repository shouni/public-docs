# Library README Convention

Decides how the READMEs of the public Go libraries are structured, and which languages the
repositories are written in.
It is for people starting a new library and people fixing an existing library's README.
After reading it you can decide mechanically what goes in the README, what is left to godoc and what is
sent to CLAUDE.md, and judge for yourself where an existing README departs from the convention.

READMEs of applications (web / worker services on Cloud Run) are covered by
[app-readme-convention.md](app-readme-convention.md). **It is a deliberately different convention**,
with a fixed skeleton (why: 3.1). The language rule in 2.7 applies to applications too.

---

## 1. Why align them

When the 19 libraries of 2026-09-04 were lined up, their READMEs ranged from 93 to 595 lines. Length
itself is down to the subject, but three kinds of rot were mixed in.

- **Copies of godoc.** One README carried a struct's field table that pkg.go.dev shows anyway. Add a
  field and there are two places to fix.
- **Tables duplicated between sibling libraries.** Two libraries with similar roles each held the same
  52-line table. Fix one and the other stays stale.
- **Three names for the same section.** Features were "Features", "Characteristics" or "Design
  highlights"; usage was "Usage" or "Quick start"; layout was "Project layout", "Package structure" or
  "Project structure". Readers had to read each time to find the section they wanted.

And only one library had the right line: gcp-kit's CLAUDE.md said "the README carries only what is
expensive not to know; signatures go to godoc, reasons to CLAUDE.md". The other 18 did not know it. A
rule in one place does not spread unless it is copied.

---

## 2. Convention

### 2.1 The three-way split

**This is the heart of the convention.** Keep it before the skeleton.

| Where | What it says | Why the reader comes |
|---|---|---|
| godoc | Signatures, arguments, return values, the meaning of zero values, the list of errors | To check how to call it |
| README | What it owns and what it does not, the minimal code for the first call, traps that bite before the reader gets to godoc | To decide whether to use it |
| CLAUDE.md | Why the boundary is where it is, rejected alternatives, holes stepped in | To change it |

There is one test: **if a sentence can be learned by opening pkg.go.dev, it does not go in the README.**
Field tables, method tables and lists of error constants are the usual cases.

**"Traps that are expensive not to know" do not come before this test.** If the trap is written in the
godoc of the symbol concerned, the README does not need it. A trap earns its place in the README only
when it **bites before the reader knows which symbol to open**.

- Keep: "retry does not look at the method, so a non-idempotent send can run twice". It decides
  whether to adopt the library at all, and has to be known before opening `Post`'s godoc.
- Drop: "`Begin` does not fall either way when it cannot read the state". That concerns people who use
  `Begin`, and it is written where `Begin` is.

To tell them apart, ask: "has the person who steps on this trap already opened the godoc of its
target?" If so, writing it there is enough.

**A one-sentence reason attached to a rule may stay in the README** ("…so an empty value is not an
error"), because it is the explanation of what is expensive not to know. CLAUDE.md gets invariants that
span several files and rejected alternatives. A reason confined to one function is closest to its doc
comment, and the README only points there.

A library without a CLAUDE.md simply has nothing of those two kinds yet. **Add one when it is needed**
(starting to pile them up in the README is the signal).

### 2.2 Skeleton

**The order and presence of headings may follow the subject.** Libraries range from one package to
eight, and forcing one shape would bend the content instead. Fixed are only the vocabulary (2.3) and
the presence of these three:

```markdown
# <emoji> <library name>
<badges>

English | [日本語](README.ja.md)

## 🚀 About - <one line: what it takes on>       ← required
(two or more packages) ## 📦 Package Structure
## 🚦 Usage                                      ← required
(a subject that cannot live in godoc) ## <that subject>
## 🤝 Dependencies
## 📜 License                                    ← required
```

Put **what it does not take on** into the one line of About as well. Saying the non-goals ("storage
is the caller's", "knows neither AI nor the cloud") lets a reader decide in a few lines.

**Subject sections are the exception.** Through the test in 2.1, most traps go to godoc. A section is
worth it only for what cannot sit on one godoc symbol: a comparison table across several functions,
the format of a file the user writes by hand, the list of what the library guarantees.

### 2.3 Heading vocabulary

Sections with the same role do not get different names. Use the left column, not the right.
Headings take the form `## <emoji> <English>`. In `README.ja.md` the heading is translated and the
English kept in parentheses (`## 🚀 概要 (About)`).

| Use | Do not use |
|---|---|
| `## 🚀 About - <one line>` | — |
| `## ✨ Features` | Characteristics, Design highlights, Core concepts, Core Features |
| `## 🚦 Usage` | Quick start, Getting started |
| `## 📦 Package Structure` | Project layout, Project structure, Packages |
| `## 🤝 Dependencies` | Main dependencies |
| `## 📜 License` | — |

"Design highlights" and "Core concepts" are not used because what goes under them is usually
CLAUDE.md material. If it cannot be written as a feature, it is a reason, not a feature.

**No installation section.** `go get <module path>` is obvious from the path and not worth a section.
If needed, put it as one line at the top of Usage.

### 2.4 Code examples

**The README carries only the first example.** One example that builds and makes one call; branches
and variations go to godoc `Example` functions. `go test` runs those, so they stop compiling when the
API changes. Nobody checks a README code block.

**Create the destination before deleting.** Delete a README example without writing an `Example` and
it is gone (one was actually lost this way). If an `Example` cannot be written, keeping it in the
README is better.

### 2.5 What not to write

| Do not write | Reason | Instead |
|---|---|---|
| Field tables, method tables, lists of error constants | godoc shows the same; adding one means two places to fix | godoc |
| The same table as a sibling library | Only one gets fixed | Keep it in one and link from the other (2.6) |
| A list of consuming repositories | It rots, and puts private names in a public document | How to find them (`grep`) |
| User counts, version numbers | They rot | [libraries.md](libraries.md) |
| Design reasons, rejected alternatives | Not needed to decide whether to use it | CLAUDE.md |
| Relative dates | See [documentation-guideline.md](documentation-guideline.md#7-leave-a-trail-of-changes) | Absolute dates (`2026-09-04`) |

### 2.6 Sibling libraries

When two libraries have neighbouring roles, **put the table that tells them apart in one of them and
link from the other.** Kept in both, one always goes stale. It goes in the later one, the one whose
readers want to know how it differs from what already exists.

### 2.7 Languages

The repositories (libraries and applications alike) are written in English, with a Japanese copy of the
README only.

| What | Language |
|---|---|
| `README.md` | English |
| `README.ja.md` | Japanese copy of `README.md`, same structure, **changed in the same commit**. Both carry a language switch under the badges: `English \| [日本語](README.ja.md)` in `README.md`, `[English](README.md) \| 日本語` in `README.ja.md` |
| `CLAUDE.md` | English, with no Japanese copy |
| Code comments, doc comments, error and log messages | English |
| Test names and subtest names | English |
| Data | Japanese stays only where it is the data itself: prompts, end-user UI text, fixtures, and test case names that are themselves input data |

### 2.8 Badges

One line per badge, in this order. The repositories agreed on it before it was written down; it is
written here so the next repository does not have to infer it.

| Order | Badge | Libraries (Go) | Applications | Python tools |
|---|---|---|---|---|
| 1 | CI (the `ci.yml` workflow, no branch query) | ✓ | ✓ | ✓ |
| 2 | Status | ✓ | ✓ | ✓ |
| 3 | Language | Go | Go | Python |
| 4 | Version or platform | Go Version (from `go.mod`) | Platform | Python Version (the oldest the pinned dependencies support) |
| 5 | GitHub tag (latest by date) | ✓ | — | ✓ |
| 6 | License | ✓ | ✓ | ✓ |
| 7 | Reference | Go Reference (pkg.go.dev) | — | — |

**A badge states only what is checked.** The Python Version badge once said 3.11+ while the pinned
numpy and pysubs2 needed 3.12; the CI matrix now runs the oldest version the badge names, so the two
cannot drift apart again. A repository without CI (such as a configuration-only one) leaves out the
CI badge rather than pointing at nothing.

---

## 3. Rejected alternatives

### 3.1 Forcing the application skeleton on libraries

```text
Adopted: fix only the vocabulary and leave the section layout to the subject
Reason: the applications all have nearly the same shape (web/worker sides, Cloud
        Tasks, environment variables), so their skeleton could be fixed. Libraries
        mix one-package and eight-package modules, and forcing one section layout
        would mean inventing content to fill sections that do not exist
Revisit when: never
```

### 3.2 Moving every code example out of the README into godoc

```text
Adopted: keep only the first example in the README
Reason: the README is what GitHub opens first. Without one working example there,
        deciding whether to adopt the library takes a round trip to pkg.go.dev. The
        problem is unchecked examples multiplying, so do not multiply them
Revisit when: never
```

### 3.3 Japanese only, or Japanese and English side by side in one README

```text
Adopted: English README.md and a Japanese README.ja.md, as two files changed in the
        same commit (since 2026-11-09)
Reason: the repositories moved to English on 2026-11-09/10. Mixing both languages in
        one file doubles every section, which is why it was rejected on 2026-09-04:
        one half gets fixed and the other does not. Two whole files keep each one
        readable from top to bottom; the risk of one going stale is handled by
        changing them in the same commit
Revisit when: none recorded
```

---

## 4. Finding READMEs that depart from the convention

```bash
# Heading vocabulary
grep -n '^## ' README.md README.ja.md
# The language switch is in both files
grep -c 'README.ja.md' README.md; grep -c '(README.md)' README.ja.md
# Japanese left in README.md (outside quoted data)
perl -CSD -ne 'print "$.: $_" if /[\p{Hiragana}\p{Katakana}\p{Han}]/' README.md
# The two copies have the same number of sections
diff <(grep -c '^## ' README.md) <(grep -c '^## ' README.ja.md)
# Badge order (2.8): compare with the table
grep -oE '^\[!\[[^]]+' README.md | sed 's/\[!\[//'
# Copies of godoc (field tables, method tables)
# Note: also matches package tables and field definitions of files the user writes by hand; read the table to judge
grep -n '^| `' README.md
# Long reasons (rejected alternatives, history) left in the README
# Note: a one-sentence reason attached to a rule is normal, so this matches a lot; read and judge
grep -n -i 'used to\|previously\|rejected\|instead of' README.md
# User counts or lists of consuming repositories
# Note: version numbers in migration notices may stay
grep -n -i -E '[0-9]+ (repositories|repos)|used by' README.md
```

**The check that matters most cannot be written with grep.** The test in 2.1 (can it be learned by
opening godoc?) can only be checked by actually opening `go doc`. The same content often hides behind
different wording. The steps:

1. List sections outside the skeleton with `grep -n '^## '`.
2. Take each claim in a section and identify the symbol it is about.
3. **Actually open** `go doc <package> <symbol>`.
4. If it is there, drop it from the README. If not, keep it.

The first application skipped this, and one README was left with 44% copied from godoc.

---

## Changelog

- 2026-11-11: Added 2.8 Badges. The order the Go repositories already shared (CI, Status, Language,
  version or platform, tag, License, Go Reference) is now written down, with the Python tools in it.
- 2026-09-04: First version. The READMEs of 19 libraries were compared, and the three-way split (godoc
  / README / CLAUDE.md) that only one of them had became the convention for all. The skeleton is not
  fixed; only the heading vocabulary and "what not to write" are. Applying it the same day loosened two
  points: the "do not use" column gained forms that actually existed (Core concepts, English-only headings),
  and CLAUDE.md became "add it when needed" rather than required (a one-sentence reason attached to a
  rule may stay in the README, and a reason confined to one function is closest to its doc comment).
- 2026-09-04: "Keep traps that are expensive not to know in the README" was ranked below the three-way
  split. When the two conflict and the former wins, the trap section becomes a copy of godoc (44% in one
  README). The test is "has the person who steps on this trap already opened its godoc?" Section 4 also
  gained: create an `Example` before deleting an example (one was lost), and this test cannot be
  checked with grep.
- 2026-11-10: Translated to English. The repositories moved to English, so README.md is English with a
  Japanese README.ja.md (3.3 reversed), headings are English, and 2.7 records the language of
  READMEs, CLAUDE.md, comments, messages and test names.
