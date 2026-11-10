# URL Naming Convention

Decides how HTTP paths are cut in the web applications on Cloud Run.
It is for people starting a new app and people adding routes to an existing one.
After reading it you can derive "what is the path for this resource" from the convention mechanically,
and judge for yourself where an existing app departs from it.

Every app is written with `net/http`-compatible handlers on [chi](https://github.com/go-chi/chi).
Naming drifted not because the router lacked expressiveness but because the convention was never
written down, so it applies without changing the router or adopting a framework.

The convention started from what most of the five existing apps already did, checked against Google's
API design guide (AIP), the Microsoft REST API Guidelines and the Zalando RESTful API Guidelines.
Decisions that differ from the mainstream are in section 3, with the reason and the condition for
revisiting them.

---

## 1. Why align them

Apps are allowed to differ in pipeline style, as a difference of age. URLs are a separate matter, for
two reasons.

- **They are an external contract.** The MCP server holds these paths as string literals and absorbs
  per-service differences on its side. The more the paths diverge, the more absorbing code there is, and
  the more 404s appear when only one side changes.
- **Browsers and agents call the same URLs.** OAuth sessions (people) and OIDC (M2M) enter the same
  routes, and the representation is switched by `Accept`. Splitting paths into "for people" and "for
  machines" would break that.

---

## 2. Convention

### 2.1 Spelling

| Item | Decision |
|---|---|
| Word separator in paths | kebab-case (`/design-sheets`, `/regenerate-keyframe`) |
| Query keys and JSON keys | snake_case (`per_page`, `job_id`). Different from paths so that a JSON key can be passed as a query key with the same spelling |
| Upper case | Never |
| Trailing slash | Never (except `"/"` inside chi's `r.Route`) |
| Path parameters | camelCase, as in `{jobID}`. The same value has the same name in every app |
| `/api/` prefix | **None** (see 3.1) |
| `/v1/` prefix | **None** (see 3.5) |

### 2.2 The resource skeleton

The job is the only primary resource. From the moment it is submitted until it is deleted, running or
finished, it is addressed by the same `/jobs/{jobID}`. "History" is a list view of finished jobs, not a
separate resource.

| Purpose | Method and path | Notes |
|---|---|---|
| Top page (submission form) | `GET /` | |
| Submission form | `GET /compose`, `GET /compose/<kind>` | Screens with no JSON counterpart (2.7). Forms whose input is different in kind, such as recipe editing or cover art, go below it: `/compose/recipe`, `/compose/image` |
| Submit | `POST /jobs` | The kind is the body's `command` (`compose_recipe`, `generate_design_sheet` and so on; empty means the default). Same for forms and JSON. Response in 2.3 |
| Job list (history) | `GET /jobs` | Finished jobs by default; filter with `?state=`. Kinds that share one record and differ only in `command` (songs, drafts, cover art and so on) are filtered with `?kind=`. Paging in 2.5 |
| Job detail | `GET /jobs/{jobID}` | Always includes progress (`state` / `terminal` / `error`); once finished, the artifacts are in the same document. How, in 2.3 |
| Delete a job | `DELETE /jobs/{jobID}` | Not `POST …/delete` |
| Get an artifact | `GET /jobs/{jobID}/<artifact>` | `/audio`, `/video`, `/script`, `/recipe` and so on. `409` while unfinished |
| Edit an artifact | `PUT /jobs/{jobID}/<artifact>` | Full replacement, hence `PUT`. No partial updates |
| Action on a job | `POST /jobs/{jobID}/<verb>` | Verbs in 2.4. Response in 2.3 |
| Child elements | `/jobs/{jobID}/<plural>/{index}` | E.g. `/cuts/{cutIndex}/keyframe`. Actions on a child take the same form: `/cuts/{cutIndex}/regenerate-keyframe`. Conditions below |
| List of options | `GET /modes`, `GET /speakers` | Plural nouns |
| Detail of an option | `GET /modes/{mode}` | In the path, not the query |
| Worker endpoint | `POST /tasks/<verb>` | The value is one constant in `internal/domain/routes.go` |
| Health check | `gcp-kit/cloudrun`'s `HealthPath` | The app does not spell the string |
| Authentication | `/auth/*` | Owned as a whole by `gcp-kit/auth` |
| Static files | `/static/*` | |

Primary resources other than jobs (characters and so on) follow the same skeleton:
`/characters`, `/characters/{characterID}`, `/characters/{characterID}/<artifact>`.

**When a child may be addressed by index.** An index such as `{cutIndex}` is allowed only when the
order no longer changes once the parent artifact is final. If insertion, deletion or reordering can
happen, switch to a stable ID (`{cutID}`). The mainstream guides ask for IDs from the start, but with
today's subject, where cuts are fixed once the script is final, an index reads better and needs no ID
minting, so it is allowed under that condition.

### 2.3 Responses to asynchronous work

Submissions and actions start a job and return at once. Responses are aligned as follows.

| Item | Decision |
|---|---|
| Status | `202 Accepted` |
| `Location` header | `/jobs/{jobID}`. The client gets the polling target without reading the body |
| JSON body | Includes `job_id`. A page may simply move to the job detail page |
| Polling | `GET /jobs/{jobID}` until `terminal` is true (failure is terminal too). After completion the same URL is the detail |

`Location` is what the mainstream guides require (Google AIP-151, Microsoft's long-running operations);
all five apps set it on the `202` of submissions and actions (2026-09-04).

**One document after completion.** The JSON for `GET /jobs/{jobID}` puts the progress fields flat at
the top level and adds the artifacts to that one document. There are two ways to add them, decided by
whether the JSON keys collide.

| How | When | Examples |
|---|---|---|
| Added flat | The artifact's keys do not overlap the progress keys (`job_id` / `state` / `title` and so on) | A song's `recipe` / `assessment` / `audio_url`, a review's `report_url` |
| Nested under one key | The artifact has its own `job_id` or `title` | `detail` (video history detail), `comic` (the comic state), `image` (cover art) |

Embedding something that cannot be added flat makes Go's `encoding/json` drop both keys of the same
name. Only a few KB go in the body; larger artifacts (a full report and so on) are linked
(`report_url`) and fetched from `GET /jobs/{jobID}/<artifact>`.

### 2.4 Action verbs

The verbs in `POST /jobs/{jobID}/<verb>` are aligned as follows.

| Verb | Meaning |
|---|---|
| `regenerate` | Make it again from the same input |
| `regenerate-<target>` | Make part of it again (`regenerate-keyframe`, `regenerate-zip`) |
| `generate-<target>` | Make the next stage from an existing artifact (`generate-video`) |
| `generate` | Make an artifact of the same kind, as a new job, from the recipe of the source (a draft or a published song) |
| `synthesize` | Make audio from a saved script (without making a script) |
| `finalize` | Combine artifacts made in pieces into one |
| `preview` | See the result without side effects, in the form `POST …/<target or verb>/preview` (`/reading/preview`, `/post-youtube-comment/preview`) |
| `apply-<target>` | Apply something to a saved asset afterwards (`apply-title`) |
| `post-<target>` | Post to an outside service (`post-youtube-comment`). Irreversible, so it always has `…/preview` |

Before adding a verb, confirm that no existing verb expresses it. Action names start with a verb. A
bare noun (`cover-art`, `title`, `youtube`) leaves the reader to guess whether it makes, applies or
posts, so it is not used. This is distinct from a sub-resource created by `POST` (`POST /jobs`): every
`POST` under a job is an action.

### 2.5 Paging and filtering lists

| Item | Decision |
|---|---|
| Page selection | `?page=N` (1-based) and `?per_page=N`. Below 1 means page 1; past the last page rounds to the last page (`go-utils/paging.New`) |
| Filters | `?<snake_case>=value`. Job state is `state` (`queued` / `running` / `succeeded` / `failed`), stage is `stage`, kind is `kind` |
| JSON shape | `items` side by side with `go-utils/paging.PageMeta` (`page`, `per_page`, `total`, `has_next` and so on) |

Cursors are not used. Behind each list is a simple descending scan of object storage or Firestore, and
the history is a few hundred entries per user. Revisit when a list takes more than a second, or when
the histories of several users are mixed into one list.

### 2.6 Error responses

| Item | Decision |
|---|---|
| JSON shape | `{"error": "<message>"}`, through `go-serve-kit/respond`'s `Error` |
| HTML | A page, according to `Accept`; `respond` decides that too |
| 4xx | Bad input `400`, unauthenticated `401`, forbidden `403`, missing `404`, duplicate submission `409` |
| 5xx | Upstream API failure `502`, anything else `500`. Details go to the log, not the body |

RFC 9457 (Problem Details) is not used (3.6).

### 2.7 HTML and JSON

The same path switches between HTML and JSON by `Accept` (`go-serve-kit/respond`'s `WantsJSON`).
A JSON-only path is cut only when:

- there is no JSON counterpart, as with the submission form (`GET /compose`), or
- **the content differs**, as with a list whose HTML is a summary and whose JSON is complete.

"It is JSON, so it gets `/api/`" is not a reason.

---

## 3. Rejected alternatives

### 3.1 An `/api/` prefix

```text
Adopted: no prefix
Reason: it conflicts with switching one URL by Accept. With /api/ there are two
        routes, an HTML one and a JSON one, so authentication, validation and loading
        are either written twice or one side becomes a thin wrapper; either way one
        side gets fixed and the two drift apart.
        The mainstream guides assume the API sits on a separate path or host because
        it is published to third parties. While the only caller is the owner's own MCP
        server, Vary: Accept avoids the caching trap.
Revisit when: the API is published to third parties, or browser and M2M callers
        need to differ not only in authentication but in rate limits or billing units
```

### 3.2 `POST …/delete`

```text
Adopted: DELETE /jobs/{jobID}
Reason: HTML forms cannot send DELETE, so some implementations kept POST …/delete
        for pages. That made two deletes, DELETE for M2M and POST for pages, with
        separate entry points for authorization and logging.
        Most apps (4/5) already had the delete button send DELETE with fetch;
        aligning on that leaves one route.
Revisit when: pages must work without JavaScript
```

### 3.3 Two roots: progress at `/jobs/{jobID}`, detail at `/history/{jobID}`

```text
Adopted: /jobs/{jobID} only
Reason: when the convention was first written, 3 of the 5 apps had two roots,
        progress at /jobs/{id} and the finished detail and artifacts at
        /history/{id}. One jobID had two URLs, so clients had to pick one by looking
        at state, and the MCP server carried that branching.
        Every mainstream guide has one resource per entity and makes history a view
        such as GET /jobs?state=succeeded. Aligned on that, a job has one URL from
        submission to deletion, and polling turns into the detail once it finishes.
        The migration was paid for with the steps in section 4.
Revisit when: never. The only reason for two roots was "that is how it was"
```

### 3.4 Creating with `POST /compose`

```text
Adopted: POST /jobs, with the kind in the body's command
Reason: the apps used to submit with POST /compose and POST /compose-<kind>.
        3.3 made /jobs the primary resource, so creation goes under the same root.
        Per-kind paths such as /compose-draft grow both the paths and the MCP server
        with every kind; one body field grows only the schema.
        The GET /compose form stays (a screen with no JSON counterpart, 2.7).
Revisit when: the input schemas of the kinds become entirely different and one
        POST /jobs turns validation into a mass of branches. Then cut a separate
        resource (/drafts and so on), not POST /jobs/<kind>
```

### 3.5 Versioning (`/v1/`)

```text
Adopted: none
Reason: every mainstream guide carries a version in the URL or a header, because it
        cannot force callers to update. Here the only caller is the owner's MCP
        server and the deployment order is the owner's to choose. Instead of
        versions, migrate with "register old and new → switch → remove the old"
        (section 4).
Revisit when: the API is published to third parties (as in 3.1)
```

### 3.6 RFC 9457 Problem Details

```text
Adopted: keep {"error": "<message>"}
Reason: Problem Details (type / title / status / detail / instance) assumes clients
        branch on the error type mechanically. Today's callers only show the message
        to a person and do not branch on type. All five apps already agree on
        {"error"}, and changing it would not pay for itself.
Revisit when: a client needs to behave differently by error type (deciding whether
        to retry and so on). Then add status and type
```

### 3.7 Addressing a mode's detail as `/modes/{mode}` (the music app)

```text
Adopted: keep GET /modes/detail?kind=&key= (an exception to 2.2)
Reason: mode keys contain a slash, as in "en/edm_pop", which does not fit a chi path
        parameter, and %2F does not mix with CleanPath's normalisation. kind
        (lyrics / compose / coverart) is needed too, so the path would be two levels.
Revisit when: the slash leaves the key (language becomes a separate axis)
```

### 3.8 Adopting a framework

```text
Adopted: stay on chi
Reason: the naming drift came from having no convention; the router is expressive
        enough. A framework that is not net/http-compatible would mean rewriting the
        gcp-kit / go-serve-kit middleware and would do nothing for naming.
Revisit when: never for naming reasons
```

---

## 4. Finding routes that depart from the convention

The five apps and the MCP server were aligned on 2026-09-04, and no old paths remain. From then on,
each new route is checked with these greps:

```bash
# snake_case in paths
grep -rn 'r\.\(Get\|Post\|Put\|Delete\)("[^"]*_' internal/server/router.go
# /api/ prefix, the /history root, submissions to /compose
grep -rn '"/api\|"/history\|r\.Post("/compose' internal/server/router.go
# 202 responses without Location
# Note: also matches the 202 of deletes and handlers that only render an HTML acceptance page; look only at JSON responses to submissions and actions
grep -rln --include='*.go' 'StatusAccepted' internal/server/handlers | grep -v _test | xargs grep -L 'Location'
```

On the MCP server side, a test checks that the backend routes each tool declares (`endpoints`) exist
in that backend's `router.go` (it runs only where the five repositories are checked out side by side).
Move a route and that test fails first.

For a large move, use the same steps as the first migration: register both old and new (pull the
routes under `/jobs/{jobID}` into a `func(r chi.Router)` and mount it at both places with `r.Route`),
switch the MCP server, and remove the old after deploying. While the owner is the only user, the double
registration may be skipped in favour of deploying both sides together.

---

## Changelog

- 2026-09-03: First version. The route registrations of the five apps were compared and the majority
  form (3/5) became the convention. The same day, it was compared with the Google AIP, Microsoft and
  Zalando guides, adding asynchronous responses (`Location`), paging, error responses, and the reasons
  for not adopting versioning or Problem Details. The majority's two roots, `/jobs` and `/history`, were
  merged into `/jobs`, and `POST /compose` became `POST /jobs`. One public app was moved first that day
  (old paths kept).
- 2026-09-04: The five apps and the MCP server finished migrating. Decisions made during
  implementation: the kind is the body's `command` (renamed from `stage`), `?kind=` on lists,
  `/compose/<kind>` for derived forms, the one document of `GET /jobs/{jobID}` (when to add flat and
  when to nest), the verbs `generate` / `synthesize`, `Location` marked as implemented, the exception
  (3.7 `/modes/detail`), and the verbs `apply-` / `post-`. Three noun-only action names (`cover-art` /
  `title` / `youtube`) were renamed the same day and are not kept in the convention. All old paths were
  removed, so section 4 changed from "how to migrate" to "how to find".
- 2026-11-10: Translated to English.
