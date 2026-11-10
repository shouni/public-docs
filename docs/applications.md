# Application and MCP Server Reference

The applications that combine the libraries in [libraries.md](libraries.md) and run on Cloud Run /
Cloud Tasks, and the MCP servers that are the entry point for AI agents.

It is here to look up **what each app keeps for itself and what it delegates to the libraries**.
Rules shared by every app are in the conventions ([URL naming](url-naming-convention.md),
[workers](worker-convention.md)) and are not repeated here. The overall design is in
"[The whole picture of a personal development ecosystem](https://zenn.dev/snknsk/articles/shouni-go-ecosystem-overview)"
(Japanese, like every article linked from this page).

---

## 🚀 Applications

### AP Music Poc

> **Archived.** It is a PoC meant to show a design, so it was deliberately frozen as a snapshot.
> The code can still be read.

A **PoC** (technical demonstration) that turns a Lyria music-generation pipeline into a web application
on Cloud Run and Cloud Tasks. It is not a product, so the prompt templates and quality control are
deliberately simplified; the priority is that the asynchronous skeleton reads plainly.

* **Asynchronous workflow**: from user input to a song blueprint (MusicRecipe JSON) → audio
  generation → MP3 and recipe JSON saved to GCS → signed URL → Slack notification, run as a job through
  Cloud Tasks. Audio is stored as MP3 to match the Lyria API's default output.
* **Architecture**: hexagonal, with the AI APIs kept outside the application boundary. Authentication,
  sessions, CSRF and OIDC have no implementation in the app; they are `gcp-kit`'s `auth` as is.
* **History and playback**: a protected UI lists the history on GCS, shows details, previews audio
  through signed URLs and deletes entries.

How the asynchronous workflow is assembled is in
"[Orchestrating an AI music generation API asynchronously with Cloud Run and Cloud Tasks](https://zenn.dev/snknsk/articles/ai-music-orchestrator)".

[GitHub - shouni/ap-music-poc](https://github.com/shouni/ap-music-poc)

---

### AP MV

A web application that makes a music video from the Music Recipe (song-structure JSON) written by the
music service. Gemini writes the cut-by-cut script (Video Recipe), a keyframe image is generated per cut,
Veo makes each cut's video, and ffmpeg joins them. Structuring the cuts and handing them to Veo is
delegated to `go-veo-orchestrator`; this app narrows itself to **job management, asynchronous
execution, history and the API/MCP surface**.

* **One cut per task; the worker enqueues the rest itself**: a Cloud Tasks request has a time limit,
  and a long MV does not fit in one dispatch. After each cut the remainder goes back in as a
  continuation task, each with its own `PIPELINE_TIMEOUT`. **The record stays `running` and no
  notification is sent**: marking it `succeeded` would trip the re-run guard on the continuation tasks,
  which share the job ID. Continuation tasks have deterministic names (`<jobID>-cont-cut-<n>`), so an
  at-least-once redelivery cannot fork the chain in two ([task names](worker-convention.md#23-task-names-deduplication)).
* **Retries are off on purpose** (`max_attempts = 1`, see [the worker convention](worker-convention.md#34-max_attempts-of-2-or-more)):
  here the extra reason is that **every regenerated cut is billed by Veo again**. Generated cuts are
  persisted, so resubmitting the same recipe resumes where it stopped.
* **Signed URLs are not embedded in HTML**: pages point at same-origin paths and the handler 302s to a
  fresh signature each time. Embedding has two problems: a signed URL works outside the app's
  authentication while it is valid, and **the detail page is where cuts are compared before deciding
  what to regenerate, so it stays open until the signatures expire**. It also removes the IAM SignBlob
  round trips one detail view used to pay up front (Cloud Run holds no private key, so signing is a
  network call, not a local computation). **Only the JSON representation returns signed URLs
  themselves**, because the MCP gateway takes the URL rather than following a redirect.
* **A separate ledger of what was billed**: the finished video's metadata cannot tell how many cuts were
  actually rendered, because cuts rendered by redeliveries or individual regenerations do not appear
  there. Right after each successful generation (the moment billing happens) the count, seconds and
  model go to `veo_usage.json`, and **the difference from the finished work is shown in the history as
  regeneration waste**. A failed write only warns and generation continues: stopping for bookkeeping
  would make Cloud Tasks re-render an already billed cut.

[GitHub - shouni/ap-mv](https://github.com/shouni/ap-mv)

---

### AP Story

A web application that generates "outline → script → design sheets → panels → pages" from a manuscript
(URL or text). The generation logic is delegated to `go-comic-kit`; this app narrows itself to **job
management, asynchronous execution, history and the API/MCP surface**.

* **Script gate**: the web form **stops at the script**. When the button is pressed the outline has not
  run, so the number of panels is unknown, and going straight to image generation could cost several
  times what was expected. Outline and script are text generation, orders of magnitude cheaper, so a
  person checks them there first.
* **One state document**: `comic_state.json` is the single source of truth; the history list, the detail
  page and regeneration all read and write it. Panel images live at **stable paths**
  (`panel_{panelID}.png`), which is what makes "redraw only panel 3 of 12" possible.
* **Deduplicating only what may be collapsed**: this app hit Cloud Tasks swallowing re-submissions as
  `ALREADY_EXISTS`; the rule it led to is in [task names](worker-convention.md#23-task-names-deduplication).

Why the script gate has two stages, and the holes around the state, are in
"[Don't start image generation before the panel count is known: designing an AI comic orchestrator](https://zenn.dev/snknsk/articles/79fa0d1bbb57e1)".

[GitHub - shouni/ap-story](https://github.com/shouni/ap-story)

---

### AP Voice

A web application that turns web articles or documents on GCS into narration with a chosen speaker and
style. Script generation is delegated to Gemini and synthesis to a VOICEVOX engine (through
`go-voicevox`); this app narrows itself to **job management, asynchronous execution, history and the
API/MCP surface**.

* **Script and audio are separate steps**: the job stops once the script exists and shows up in the
  history. To fix a reading or a speaker, **only synthesis is re-run, not generation**. Synthesis takes
  minutes, so running it on a script that is about to be fixed is wasted. It is the same decision as AP
  Story's script gate, but the reason to stop is rework, not cost.
* **The script is not in the task payload**: the synthesis task carries only the job ID
  ([large inputs](worker-convention.md#22-payload)). Scripts brought in by users take the same path, so
  there is one way to read a script.
* **The app owns the speaker vocabulary**: scripts name speakers in words (a character name plus a
  style such as "normal"), and `go-voicevox` resolves them to the engine's numeric IDs. The output
  schema given to the AI is built from the same speaker list, so **a speaker that does not exist never
  appears in a script**.
* **Readings are pinned where they can be**: proper nouns and coined words are unstable under
  morphological analysis. Per-spelling reading overrides (`audio`'s reading normalisation, used through
  `go-voicevox`) settle those first, and only the rest goes to the analyser.

[GitHub - shouni/ap-voice](https://github.com/shouni/ap-voice)

---

### ADK Review

A web application that has an **AI agent** review a Git diff. Rather than handing the diff to a model
and stopping, it runs as an agent loop on [ADK for Go](https://github.com/google/adk-go).

* **It investigates beyond the diff itself**: it has three read-only tools (read a file, list, search)
  and gathers the context it needs, starting from the diff. The target is **findings the diff alone
  cannot surface in principle**: contradictions with configuration elsewhere, breakage at the callers of
  a changed function.
* **The AI SDK exception is contained**: ADK requires the `genai` SDK directly, which breaks the rule
  that the SDK stays inside `genai-kit`. The exception is confined to the reviewer, the output schema and
  one file that builds the client configuration; `go-review-kit` knows nothing about ADK.
* **It stops by count, not by time**: the investigation is capped by the number of tool calls. A
  model's thinking time cannot be estimated at design time, but the number of look-ups needed can be
  specified. At 80% of the cap it is told "start no new investigation; conclude with what you have",
  so **it ends in a finished review rather than a deadline overrun**.
* **The working directory is treated as external input**: the repository under review comes from
  outside, so `filepath.Clean` is not enough for the tools' path checks. Paths are resolved to their real
  location, and it is confirmed that no symlink leads outside the working directory.

What changed when it became an agent is in
"[Agents read outside the diff: rebuilding AI review with ADK for Go](https://zenn.dev/snknsk/articles/adk-go-agent-review-practice)".

[GitHub - shouni/adk-review](https://github.com/shouni/adk-review)

---

## 🔌 MCP Servers

Entry points for AI agents (Claude Code and others) to operate these tools and SaaS directly.

Why the services are gathered behind a gateway, and why authentication has two stages (shared key and
OIDC), is in
"[One sentence to Claude Code: linking three Cloud Run services over MCP to publish an AI music video on YouTube](https://zenn.dev/snknsk/articles/claude-code-mcp-content-pipeline)"
(it covers the three services that existed when it was written).

### AP MCP

A gateway that gathers each service's features as MCP tools behind one endpoint, so that Claude Code can
call them in conversation. It fronts **five services** (music, MV, comics, voice and review). **The
project is private for now**, so only the design is excerpted here.

* **Tool descriptions are Markdown assets, not Go literals**: a description is prose the model reads on
  every request and is refined like a generation prompt. It lives at `{service}/{tool}.md`, and its
  front matter holds the title, the annotation class, the status tool to poll and the backend routes the
  tool calls. As string literals, there was no way to read what actually arrives after concatenation.
* **Annotations are declared, not inferred**: every tool declares `readOnlyHint` / `destructiveHint` /
  `idempotentHint` / `openWorldHint`. Client-side permission controls, such as auto-approving read-only
  tools, then work from declarations rather than guesses from tool names.
* **The confirm gate is the preview itself**: irreversible or public operations do not run until
  `confirm=true` is given, and return only a preview of what would happen. **There is no separate
  preview-only tool**: with a second route, which route the model picks would decide whether the gate
  applies.
* **Whether a gate is needed depends on the cost of undoing, not the verb**: a tool starting with
  `delete_` gets no gate when recreating its target is seconds of text generation (deleting a draft).
  Published artifacts, and artifacts that re-running would not bring back (anything that went through
  non-deterministic generation), get one.

### AP MCP Slack

An MCP server that posts, updates and deletes messages and reads various information through Slack
Incoming Webhooks and the Slack Web API. It uses the stdio transport, launched as a command by the MCP
client, so there is no server to host.

* **Tools are registered according to credentials**: only the tools whose token or webhook is configured
  appear in the list. In a token-only environment the webhook tool is not registered at all.
* **Preview is the default**: every tool that changes Slack requires `confirm`; without it, it **does not
  touch Slack at all** and returns the payload that would be sent. There is no preview-only tool, for
  the same reason as AP MCP.

The MCP design that goes "beyond an API wrapper", confirm gate included, is in
"[Don't stop at an API wrapper: a Go design that makes Slack operations safe from an MCP server](https://zenn.dev/snknsk/articles/mcp-server-is-not-api-wrapper)".

[GitHub - shouni/ap-mcp-slack](https://github.com/shouni/ap-mcp-slack)
