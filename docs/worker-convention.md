# Worker Convention

Decides how the applications whose web side enqueues to Cloud Tasks, and whose worker side receives the
task and makes the artifact, handle the "life" of one job.
It is for people starting a new worker app and people adding a command to an existing one.
After reading it you can derive "what happens, in what order, from submission to recording the outcome"
from the convention mechanically, and judge for yourself where an existing app departs from it.

How URLs are cut is in [url-naming-convention.md](url-naming-convention.md). Infrastructure-side decisions
about queues and service timeouts are in the infrastructure repository's conventions (private). This
document covers what sits between them: the part the application's code owns.

---

## 1. Why align them

Every incident in which a job silently disappeared came from mixing up an **order** or a **context**.

- Recording queued after enqueueing: a worker reached within tens of milliseconds wrote running, and the
  late queued overwrote it (two apps hit this in production).
- Recording the failure path on a detached ctx but the success path on the raw ctx: jobs that finished
  around the deadline stuck at running (found in three apps at once).
- Not recovering a panic: the HTTP side just returned 500 and no outcome was recorded. With
  `max_attempts = 1` no retry came either, and the job stayed running forever (only one app recovered).

None of these is a typo; each is a fix applied to one path and forgotten on the other. Shared helpers
cannot prevent that. A library that owns the order itself makes the forgetful version impossible to
write. That library is `gcp-kit/worker.Lifecycle`, and this convention is the agreement on how to fill
it in.

---

## 2. Convention

### 2.1 The life of a job

```
Web side                               Worker side (gcp-kit/worker.Lifecycle)
────────────────────────────────       ──────────────────────────────────────────
1. Mint the job ID                     4. Prepare   put log correlation keys on ctx
2. Record queued (Store.Save)          5. Begin     redelivery guard + record running
3. Enqueue to Cloud Tasks              6. Validate  check the input (failure is Permanent)
   → 202, Location: /jobs/{jobID}      7. Run       the body (the only place Timeout applies)
                                       8. Finish    record the outcome, notify (detached ctx)
```

| Step | Decision | Reason |
|---|---|---|
| 2 → 3 | **Record queued, then enqueue** | The other way round, the late queued overwrites the running the worker wrote. If enqueueing fails, undo queued |
| 5 | Use `Recorder.Begin`: read the previous record once, stop if it already succeeded, otherwise record running | With `AlreadySucceeded` and `Record` as separate calls, another delivery can slip in between |
| 5 fails | **If the state cannot be read, do not run.** Notify without a record if a notifier exists, and return the error | Falling to "unfinished" rebuilds finished jobs; falling to "finished" ACKs an unfinished job that never runs again. Fall to neither and tell a person |
| 5 → 6 | Begin before validation | Every attempt shows up in Attempts, and a job rejected by validation still goes through running to failed |
| 6 | `Lifecycle` wraps a `Validate` failure in `worker.Permanent` itself. A failure inside Run that redelivery cannot fix (a missing reference, a limit exceeded) is wrapped by the app | Unwrapped, it becomes a 500, is redelivered, and the same failure is notified twice. Wire it even with `max_attempts = 1`, so nothing has to be hunted down when the limit is raised |
| 7 | The run-time limit is `Lifecycle.Timeout` (`PIPELINE_TIMEOUT`), applied to Run only | Applied to the caller's ctx, the Finish right after the cut-off would also run on an expired ctx. If the library Run calls already bounds its own run, do not set it twice (adk-review leaves it to `go-review-kit`'s `pipeline.WithRunTimeout`) |
| 7, the ladder | **`PIPELINE_TIMEOUT` < `TASK_DISPATCH_DEADLINE` ≤ the Cloud Run request timeout.** The app checks the first inequality at startup and rejects equality and "unlimited"; neither value has a default. The second is checked on the infrastructure side | **The app has to give up first.** If Cloud Tasks cuts the request, the process stops mid-job, neither the failure record nor the notification runs, nothing is redelivered with `max_attempts = 1`, and the job stays running forever. No defaults because the ladder is the deployment's decision: an app-side default puts the number in two places and lets a missing setting run on a value nobody chose |
| 7, panic | The library recovers it and hands `worker.ErrPanicked` to Finish | No `recover` in the app, so there is no place to forget one |
| 8 | Success and failure go through the same Finish, record then notify | Do not tell a person about a success that was not recorded |
| 8, ctx | The library detaches it with `context.WithoutCancel` and bounds it (`DefaultFinishTimeout`, 30 seconds) | When the cut-off is the reason for ending, the original ctx is already done |
| 8, on failure | Finish returns the cause as is: Permanent stops with 2xx, anything else is a 500 and redelivered | Recording and notification are done, so what the user sees does not change |

Finish is not called only when Begin returns "already succeeded" or "cannot read".

### 2.2 Payload

| Item | Decision |
|---|---|
| Type | The enqueuer (`gcp-kit/tasks.Enqueuer[T]`) and the receiver (`worker.Handler[T]`) share the same `domain.Task`. The contract is the type; no JSON copy in two places |
| `job_id` | Required. It becomes a GCS path and a Firestore document ID, so validate it with `go-utils/jobid` before use |
| `command` | The kind of job. Present even in an app with one command; status records and list filters (`?kind=`) read it |
| Large inputs | Not in the payload. Save first and pass only the ID (the 1 MB Cloud Tasks limit; scripts, recipes) |
| Endpoint | `POST /tasks/<verb>`. The path is one constant in `internal/domain/routes.go`, read by both the enqueuer and the receiver |

### 2.3 Task names (deduplication)

Give a task a deterministic name (`EnqueueWithName` / `WithTaskID`) only **when a second message with
the same name is "the same work"**. Cloud Tasks keeps deduplicating for a while after completion, so a
re-submission under the same name is silently discarded as `ALREADY_EXISTS`.

- Name it: tasks that continue the same job (per-cut continuation). Running them twice is wrong.
- Do not name it (or put the submission time in the name): rebuilds and regenerations. Re-submitting is
  the correct operation.

Naming a task after its target alone lets a rebuild be absorbed by the previous, completed task.
ap-story hit this and narrowed naming to the duplicates that may be collapsed.

### 2.4 Where state is kept

Firestore (`gcp-kit/jobstatus`) and GCS (`go-job-kit/jobstatus`) are both allowed. `Recorder` has the
same shape (`Begin` / `Record`) and Lifecycle does not tell them apart. How to choose is in
[libraries.md](libraries.md#gcp-kit): by the cost of listing, not by atomicity.

### 2.5 Files and names

| Item | Decision |
|---|---|
| Package | `internal/pipeline`. The worker's body does not go in adapters (adapters are only the boundary with outside services) |
| Entry | `pipeline.go` holds the `Runner` type and `NewRunner`, which assembles and returns the `Lifecycle` |
| Unit of work | `Step` (`Name()` and `Execute(ctx, *Context)`). Values passed between steps ride on the one `Context`, not on arguments |
| Step files | One step per file, `step_<name>.go`. When there are many steps and the package swells, move them to a `step/` subpackage without the prefix (`step/scene_split.go`) |
| Command → steps | Only `planner.go`'s `Planner.Plan(task) ([]Step, error)` holds the mapping, even in an app with one step per command. `Runner.run` runs the returned steps in order and does not branch on `command`. An app whose body is one library call with no steps (adk-review) has neither `Step` nor `planner.go` and calls it directly from `Run` |
| Status records | `job_status.go` (`begin` / `markSucceeded` / `markFailed`) |
| Notification | Called from Finish after the record. No fixed file: most apps keep it in `pipeline.go`, ap-voice in `notifier.go` |
| Type names | Do not repeat the app name (not `AdkReviewRunner`). The package (`internal/pipeline`) already names the app |

The steps used to be called differently depending on when the app was written (Filter / Workflow /
Step); they were aligned on `Step`. Telling kinds apart by prefix was considered, but it makes grep take
three forms and leaves whoever starts the next app unsure which to copy; one name is cheaper.

---

## 3. Rejected alternatives

### 3.1 Cutting the run time in `worker.Handler`

```text
Adopted: Lifecycle.Timeout (Run only). Handler's WithTimeout was removed
Reason: the Handler's limit covers the whole executor, so recording the outcome also
        runs on an expired ctx. The record is needed exactly when the run was cut
        off, so the limit has to sit inside the record. With limits in two places it
        is also unclear which one fired
Revisit when: never
```

### 3.2 "Going ahead" when the state cannot be read

```text
Adopted: do not go ahead. Notify and return the error
Reason: one app went ahead, arguing that running twice is easier to recover from
        than stopping because the record could not be read. But going ahead rebuilds
        finished jobs on redelivery and pays the very cost the guard exists to
        prevent. Unreadable state is rare, and a person can resubmit
Revisit when: only for an app whose double run costs nothing. Decide it inside
        Begin and write it down as an exception to the convention
```

### 3.3 Writing `recordOutcome` per app

```text
Adopted: pass it to Lifecycle.Finish. The library owns the order and the detached ctx
Reason: five apps had copied the same 20 lines, and three of them missed detaching
        the success path. As long as it is copied, the next app will miss it too
Revisit when: never
```

### 3.4 `max_attempts` of 2 or more

```text
Adopted: 1 (AP Voice's queue alone is 2)
Reason: retries hide what happened. A failure is left in the record and the
        notification on the spot, and a person resubmits. Raising it presupposes the
        Permanent wiring (2.1, step 6). The convention asks every app for it, but not
        every app has finished, so check before raising
Revisit when: an app appears where losses from transient failures cost more than
        resubmitting by hand
```

---

## 4. Finding code that departs from the convention

```bash
# A TaskExecutor implemented without Lifecycle
grep -rn 'func (.*) Execute(ctx context.Context' internal --include='*.go' | grep -v _test
# recover / WithoutCancel left in the app (Lifecycle owns them; on the Finish path they are duplicates)
# Exception: saving an artifact mid-step (a draft or partial result worth keeping precisely when the run is cut off)
grep -rn 'recover()\|WithoutCancel' internal/pipeline --include='*.go' | grep -v _test
# Step types defined under another name (library types such as Workflows do not count)
grep -rn 'type \w*\(Filter\|Workflow\)\b' internal/pipeline --include='*.go' | grep -v _test
# Command → steps mapping outside planner.go (a switch in Runner.run is suspect)
grep -rn 'switch .*Command' internal/pipeline/pipeline.go
# Order of enqueue and the queued record (the record's line must come before Enqueue; read and judge)
grep -rn 'Enqueue(\|ecordQueued' internal/server/handlers --include='*.go' | grep -v _test
```

---

## Changelog

- 2026-09-04: First version. The worker entry points of five apps were compared, and the order they
  already shared (queued → enqueue; Begin → validate → run → record) moved into
  `gcp-kit/worker.Lifecycle` and became the convention. The three points where they differed (panic
  recovery, `Permanent` wiring, going ahead when the state cannot be read) were settled in the library
  and the convention. The names of steps (Filter / Workflow / Step) and the home of the command → steps
  mapping (`planner.go`) were aligned too.
- 2026-11-10: Translated to English. Step 6 now says that `Lifecycle` wraps `Validate` failures itself.
  The timeout ladder, enforced by all five apps, moved here from the application reference. The
  `notify.go` file name was dropped, since no app uses it. adk-review's run limit is noted as living in
  `go-review-kit`.
