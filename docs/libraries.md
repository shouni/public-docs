# Library Reference

A **boundary map** of the public Go libraries: what each library takes on and what it does not, how it
differs from its neighbours, and why it sits in its layer.

It holds only **what cannot be written from inside a single repository**. Everything else lives here:

| What you want to know | Where it is |
|---|---|
| How to use it, the first call | The repository's README |
| Signatures, arguments, errors | godoc (pkg.go.dev) |
| Design decisions inside the library, rejected alternatives | The repository's CLAUDE.md |

Libraries are listed from the bottom layer up. The overall design and how the layers relate are in
"[The whole picture of a personal development ecosystem](https://zenn.dev/snknsk/articles/shouni-go-ecosystem-overview)"
(Japanese, like every article linked from this page). Applications and MCP servers are in
[applications.md](applications.md).

Dependency counts below were taken from the `go.mod` files of the owner's repositories (private ones
included) on 2026-11-10.

---

## 🏗️ Core Infrastructure Libraries

The shared base for safe outbound traffic, retries, storage access, fetching source material, audio
binaries, notifications and asynchronous jobs. None of them knows anything about AI, so applications
that generate nothing can use them as they are.

### netarmor

Owns only the **safety** of outbound connections, and knows neither AI nor cloud services.

* **Why it is the bottom layer**: SSRF protection is something every library should write the same way,
  and it is the kind of thing that **still works when you get it wrong**. Forget to block private IPs in
  one place and nothing happens in normal use.
* **No external dependencies**: the `require` block in `go.mod` is empty and `go.sum` is 0 bytes. Many
  repositories pull this module in, so one added line would land in the module graph of repositories
  that never use it.
* **`retry` moved to go-http-kit** (v1.4.0): exponential backoff used to live here, but its only users
  were `go-http-kit` and `go-web-reader`, which already depended on `go-http-kit`. Moving it added no
  dependency edge and emptied `require`.
* **Cloud storage schemes are not its business**: `gs://` and `s3://` used to pass the validator without
  name resolution. That was dropped, because **a validator that passes URIs it never connects to makes
  "passed validation" guarantee nothing**. Scheme handling sits with `go-remote-io`, which actually reads
  and writes.

It has the widest reach in the ecosystem: it appears in the `go.mod` of 11 repositories, 7 of which
require it directly; the rest get it through `go-http-kit`.

An article covers this library alone through four pillars (security, agility, resilience,
traceability): "[NetArmor: four pillars against the chaos of AI](https://zenn.dev/snknsk/articles/netarmor-four-pillars)"
(written while `retry` still lived here).

[GitHub - shouni/netarmor](https://github.com/shouni/netarmor)

---

### go-utils

Only small pieces of code that were actually duplicated across projects. Five independent packages:
`jobid` (minting and validating job IDs), `jst` (conversion to Japan Standard Time), `paging` (listing
page metadata), `slogctx` (a slog handler that adds attributes stored on the context) and `strlist`
(normalising string lists from configuration).

A package called `utils` accepts anything, so **three conditions decide what goes in**:

* **No external dependencies**: the standard library only; `go.mod` has no `require` line.
* **No I/O or infrastructure**: networking and cloud SDKs belong to purpose-built libraries such as
  `go-remote-io` and `gcp-kit`.
* **Used by two or more projects**: something only one project uses stays in that project's
  `internal/`. What looks general usually turns out to be tied to that project's own decisions.

The third condition is the one that bites: 10 repositories require this module directly. `paging` is
here for the same reason: listings on GCS (`go-job-kit`) and on Firestore (`gcp-kit/jobstatus`) return
the same `PageMeta`, and an M2M client reads both through that one type. The base libraries `gcp-kit`,
`go-job-kit` and `go-notify` depend on it too; nothing here depends back on them.

[GitHub - shouni/go-utils](https://github.com/shouni/go-utils)

---

### go-serve-kit

Takes on only the boilerplate an HTTP service writes on the **responding** side every time: defensive
headers, picking a representation, declaring the process's role and serving embedded assets. It has no
server of its own. Four independent packages (`respond` / `secureheaders` / `serverrole` /
`staticfiles`), and **an empty `require` block**.

* **Split out of gcp-kit**: none of this depended on GCP, yet an HTTP service that used no cloud had to
  import a Cloud Run kit to get it. **The two modules do not depend on each other** in either direction.

[GitHub - shouni/go-serve-kit](https://github.com/shouni/go-serve-kit)

---

### gcp-kit

Takes on only the parts every web app on Cloud Run and every Cloud Tasks worker writes the same way:
login, sessions, enqueueing and receiving tasks, recording progress and log formatting. Its six packages
(`auth` / `cloudlog` / `cloudrun` / `jobstatus` / `tasks` / `worker`) are independent; import only what
you need.

* **It owns a job's life, order included**: `worker.Lifecycle` fixes the order "redelivery guard →
  validation → run → record the outcome". Shared helpers could not prevent the slips this fixed; owning
  the order makes them impossible to write. Why, and how apps fill it in, is in
  [worker-convention.md](worker-convention.md).
* **There are two places progress can be recorded**: `jobstatus` here is the Firestore one; the GCS one
  is in `go-job-kit`. `Recorder` has the same shape (`Begin` / `Record`) in both, and callers do not
  tell them apart. **Choose by the cost of listing, not by atomicity**: while a prefix scan is enough,
  GCS is fine; once the history needs queries to filter, move to Firestore.
* **Nothing that does not depend on GCP**: response negotiation and defensive headers went to
  `go-serve-kit`.

The Cloud Run setup this kit assumes is written up in three articles:

* Splitting one image into web and worker services, switching the dependency graph by role:
  "[Split one image into web/worker on Cloud Run and narrow permissions and settings](https://zenn.dev/snknsk/articles/cloudrun-web-worker-split)"
* The asynchronous processing that `tasks` / `worker` cover, compared in Go and Python:
  "[Comparing Cloud Run + Cloud Tasks asynchronous processing in Go and Python](https://zenn.dev/snknsk/articles/cloudrun-cloudtasks-go-python)"
* Moving environment variables and model names out of the app build:
  "[A full rebuild for every model name change: moving Cloud Run settings to Terraform](https://zenn.dev/snknsk/articles/cloudrun-config-terraform-import)"

[GitHub - shouni/gcp-kit](https://github.com/shouni/gcp-kit)

---

### go-http-kit

Two packages: a `net/http`-compatible HTTP client with `netarmor/securenet` SSRF protection and
exponential backoff built in (`httpkit`), and the general-purpose retry engine underneath it (`retry`).

* **The work is split in two**: `retry` decides only when and how long to wait; what is worth retrying
  and how to resend a request belong to `httpkit`. This keeps HTTP knowledge out of `retry`, **which is
  why `go-web-reader`'s non-HTTP fetch loop can run on the same engine**.

[GitHub - shouni/go-http-kit](https://github.com/shouni/go-http-kit)

---

### go-remote-io

An I/O library that puts Google Cloud Storage, Amazon S3 and the local file system behind one
`Open(ctx, name)`, picking the backend from the URI scheme. It is **where artifacts are kept**.

* **The core imports no cloud SDK**: the root package holds only the abstraction; the GCS SDK is used
  only by `remoteio/gcs` and the AWS SDK only by `remoteio/s3`, so an app that uses one never builds
  the other.
* **Fetching source material is `go-web-reader`'s job**: both read `gs://`, but they serve different
  stages (see the next section).

The storage design this library writes into (artifacts and metadata kept apart, delivery through signed
URLs, deleting related objects together) is in
"[Designing AI app artifacts through history, playback and deletion](https://zenn.dev/snknsk/articles/gcs_object_design)".

[GitHub - shouni/go-remote-io](https://github.com/shouni/go-remote-io)

---

### go-web-reader

Reads `https://` / `http://` / `gs://` / `s3://` through the same `Open(ctx, uri)`. It is **where
source material for the AI comes from**.

Its line with go-remote-io: go-web-reader is read-only, does not read local files, and extracts the
main content from web pages; go-remote-io reads and writes, signs URLs, lists, handles local files and
returns bytes as they are. The full comparison table lives in one place only, the
[go-web-reader README](https://github.com/shouni/go-web-reader#where-go-remote-io-ends-and-this-library-begins).

* **IP validation right before connecting is `netarmor`'s job**: since it fetches URLs users typed,
  validating the URI before the fetch is not enough. A name that resolves to an internal IP after
  validation is blocked at the bottom layer.

[GitHub - shouni/go-web-reader](https://github.com/shouni/go-web-reader)

---

### audio

Handles only audio binaries and Japanese readings. Like netarmor, it knows neither AI nor the cloud.

* **Synthesis and joining are separate libraries**: joining WAVs and normalising readings live here;
  speech synthesis is `go-voicevox`. Swapping the speech engine leaves the joining side unchanged.

The reading normalisation is the subject of
"[No misreadings from the AI: transpiling prompts into a safe reading form](https://zenn.dev/snknsk/articles/go-kagome-prompt-defense)".

[GitHub - shouni/audio](https://github.com/shouni/audio)

---

### go-job-kit

The shared base for asynchronous jobs that are submitted to Cloud Tasks and write their artifacts to
object storage. **Whether the output is music, video or comics, the skeleton "submit → record progress →
page through the history" is the same**, and copies of it in each app's `internal/` drift apart, so the
shared part was pulled out. Enqueueing itself is `gcp-kit/tasks`; this module holds the state, the
listing and its cache.

* **It does not reach into the artifacts' domain**: what gets generated is the app's concern; what is
  shared is only how state is recorded and shown as history.
* **The Firestore counterpart is `gcp-kit/jobstatus`**: how to choose is in [gcp-kit](#gcp-kit) above.
  `joblist` / `paging` / `cache` exist because object storage has no queries, so a Firestore app does
  not need them.

Operational decisions such as concurrent history loading, caching and how `job_id`s are formed are in
"[Designing AI app artifacts through history, playback and deletion](https://zenn.dev/snknsk/articles/gcs_object_design)".

[GitHub - shouni/go-job-kit](https://github.com/shouni/go-job-kit)

---

### go-notify

Owns only delivering the results of asynchronous pipelines to people. It has no CLI; it is embedded in
applications.

* **The channel sits outside the body**: bodies are built as standard Markdown, and the `slack` package
  converts them to Slack's mrkdwn. **Code that builds a body never knows the channel.**

[GitHub - shouni/go-notify](https://github.com/shouni/go-notify)

---

## 🤖 AI Abstraction Libraries

Libraries that abstract prompt building, response handling, shared character definitions and image,
audio and video generation. The orchestrators above this layer do not import the SDKs below it.

### go-prompt-kit

Manages prompts for the AI and turns responses into documents (Markdown / JSON to HTML). Four
independent packages: `prompts` / `resource` / `frontmatter` / `htmldoc`.

* **It does not interpret the metadata format**: the front matter parser is supplied by the caller, so
  this module does not depend on a YAML library. A library that fixes the parser forces its users to
  release in step with it at every switch. When `gopkg.in/yaml.v3` was archived and the users moved to
  its successor, **this module did not change a line**.

The four packages, lined up from input to output, are the subject of
"[Closing the work of adding a mode into one Markdown file: pulling prompts out of Go code](https://zenn.dev/snknsk/articles/go-prompt-kit-mode-per-file)".

[GitHub - shouni/go-prompt-kit](https://github.com/shouni/go-prompt-kit)

---

### genai-kit

**The only module in this layer that imports `google.golang.org/genai`.** It holds text generation
(`gemini`), music generation from lyrics through recipe to audio (`lyria`), video generation submission
and polling (`veo`), image generation with reference images (`imagegen`), the call guard (`callguard`)
and the shared song-structure vocabulary (`music`).

* **The backend is Vertex AI**: authentication is left to Application Default Credentials, and callers
  do not care which backend they are on. The exception is `Config.APIKey`: the latest Lyria is not
  offered on Vertex AI, so music generation alone needs the API key path. **It is temporary and goes
  once Lyria is available on Vertex AI.**
* **Fetching reference images is not this library's job**: Vertex AI resolves `gs://` on the model side,
  so nothing is transferred. When http(s) references are needed, the caller decides the route, timeout
  and size limit and passes the bytes in `imagegen.Request.References`. Abstracting fetch options inside
  the library would ship dependencies few callers use (an HTTP client, a cache, recompression) to all of
  them.
* **`MusicRecipe` is the ecosystem's common currency**: the song blueprint in the `music` package is
  **taken as input, in the same shape, by the video orchestrator too**. `music` is a leaf package with
  no dependencies, so downstream code that only reads and writes recipes does not import the workflows.
* **Rate limiting is not owned per library**: quota is per project, so the `callguard` guard is placed
  above and shared with text generation. Independent rate limits per library would add up past the quota.

[GitHub - shouni/genai-kit](https://github.com/shouni/genai-kit)

---

### go-character-kit

A small library that loads, validates and looks up characters (`id` / `name` / `seed` /
`reference_url` / `visual_cues`) defined as JSON.

* **Why it is a library of its own**: the seed and reference assets, the core of consistency, are
  **needed by both comics and video**. Put in either orchestrator, the other would have to import it.
  Five repositories, `go-comic-kit` and `go-veo-orchestrator` among them, read the same definitions.

[GitHub - shouni/go-character-kit](https://github.com/shouni/go-character-kit)

---

### go-voicevox

Takes `[]ScriptLine` and returns the combined WAV bytes, nothing more. It neither writes files nor
uploads, and depends on neither. **Where the result is saved is the caller's decision.**

* **Joining is left to `audio`** (see [audio](#audio)).

[GitHub - shouni/go-voicevox](https://github.com/shouni/go-voicevox)

---

## 🎨 Generation Orchestrators

The layer that combines the AI abstractions into one work. The domain of what is being made (the
state of the work, the order of steps, keeping it consistent) ends here; the applications above only
take requests, authenticate and run asynchronously.

### go-comic-kit

A toolkit that builds comics step by step while keeping characters consistent.

* **Every prompt comes from the app (DI required)**: the kit ships no prompts. Prompts are tuned per
  work; kept in the kit, changing a single word would need a kit release.
* **Deduplication needs two layers**: collapsing identical concurrent calls into one is `callguard`'s
  singleflight, which works **within one process**. Stopping a redelivery that arrives later is the
  consuming app's re-run guard (`Recorder.Begin`). Different layers, so both are needed.

MangaState and partial regeneration are the subject of
"[AI comic generation in Go: one state document makes 'redraw only panel 3' possible](https://zenn.dev/snknsk/articles/8896d7672c5f68)".

[GitHub - shouni/go-comic-kit](https://github.com/shouni/go-comic-kit)

---

### go-veo-orchestrator

A backend orchestrator that structures a **Music Recipe** into a sequence of video cuts and hands them
to Google's video generation AI, **Veo**. AI calls go through `genai-kit`.

* **Its input is the same recipe the music side writes**: it fills in durations and start and end
  positions from the `MusicRecipe`'s `sections` and `cuts`. Music and video read the same shape, so
  nothing converts between them.
* **It contains no Veo client**: the actual call appears only as `ports.VideoRunner`, a one-method
  contract, and **the implementation is not in this repository** (the caller injects it).
  Orchestration, keyframe generation and metadata storage are separate concerns.

[GitHub - shouni/go-veo-orchestrator](https://github.com/shouni/go-veo-orchestrator)

---

### go-review-kit

Takes the diff between two Git references, has an AI review it, then saves the result and notifies,
as one pipeline.

* **No AI SDK**: unusually for this ecosystem, **it bundles no AI client at all**. The reviewer is a
  port, `WorkspaceReviewer`, and the consumer plugs in the implementation. Which SDK reviews is the
  app's choice; a library holding one would push that dependency onto every user. Its only direct
  dependency is `go-git`.

What plugging in an agent-style reviewer looks like is in
"[Agents read outside the diff: rebuilding AI review with ADK for Go](https://zenn.dev/snknsk/articles/adk-go-agent-review-practice)".

[GitHub - shouni/go-review-kit](https://github.com/shouni/go-review-kit)
