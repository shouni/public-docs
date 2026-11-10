# public-docs

English | [日本語](README.ja.md)

A home for documentation that is fine to publish and share.

What it holds today is the **reference** for a personal ecosystem (public Go libraries and the
Cloud Run applications built on them) and the **conventions** several repositories have to agree on.

## Documents

### Reference

| Path | Description |
| --- | --- |
| [docs/libraries.md](docs/libraries.md) | Boundary map of the public Go libraries (what each one owns, where it stops, which layer it sits in) |
| [docs/applications.md](docs/applications.md) | Reference for the applications and MCP servers |

### Conventions

| Path | Description |
| --- | --- |
| [docs/url-naming-convention.md](docs/url-naming-convention.md) | URL naming for the web applications (the resource skeleton, rejected alternatives, how to find routes that drift) |
| [docs/worker-convention.md](docs/worker-convention.md) | Cloud Tasks workers (the order of a job's life, payloads, task names, rejected alternatives) |
| [docs/app-readme-convention.md](docs/app-readme-convention.md) | Application READMEs (README vs CLAUDE.md, the fixed skeleton, what not to write) |
| [docs/library-readme-convention.md](docs/library-readme-convention.md) | Library READMEs (the godoc / README / CLAUDE.md split, heading vocabulary, what not to write, languages) |

### Writing

| Path | Description |
| --- | --- |
| [docs/documentation-guideline.md](docs/documentation-guideline.md) | How documents in this repository are written (structure, recording decisions, pre-publication checks) |

## What goes here and what does not

**When in doubt, ask whether several repositories refer to it.** An explanation only one repository
uses stays fresher in that repository, where it is fixed together with the code. Usage belongs in each
repository's README, the API in godoc (pkg.go.dev), and internal design decisions in each repository's
CLAUDE.md.

**This is a public repository.** No private repository names, credentials or personal information.
The writing rules and the pre-publication checklist are in
[documentation-guideline.md](docs/documentation-guideline.md).

The documents under `docs/` are written in English. Only this README has a Japanese copy
(`README.ja.md`), changed in the same commit as `README.md`.

When you add a document, add a row to the tables above as well. This checks that the tables and
the files match:

```bash
diff <(grep -o 'docs/[a-z-]*\.md' README.md | sort -u) <(ls -1 docs/ | sed 's|^|docs/|' | sort)
```

## License

Copyright of the documents belongs to the author.
Where terms of use need to be stated, put them in a separate license file or in the document itself.
