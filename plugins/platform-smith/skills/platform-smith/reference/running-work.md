# Running work: launch → session → events → artifacts

## 0. The credential pre-flight (do this first, every time)

A human using the web launcher sees unusable options greyed out. **You have no such affordance, and
this check is your equivalent of a disabled row.**

Agent definitions may carry **`harness_credential_configured`** (boolean):

| Value | Meaning |
|---|---|
| `true` | Startable. |
| `false` | The coding agent has **no usable credential**. The start is **not refused** — it succeeds, and the agent fails at its **first turn** (there is no fallback key). Do not offer it, do not pick it. |
| **absent** | **UNKNOWN — not available.** Never read absence as "fine". |

Absence is legitimate in exactly two cases, because availability is per-caller **and** per-workspace
and these calls cannot answer honestly:

- `get_agent_definition` **without `workspace_uuid`** — pass one and the field appears.
- `list_scoped_agent_definitions` at `scope=company` without `workspace_uuid`.

Resolve an UNKNOWN with **`get_readiness`** for the target — one call, an unambiguous chain
(`NEEDS_CODING_AGENT_CREDENTIAL` plus the other prerequisite steps), and it answers this question
correctly. (`get_agent_definition_secret_refs_status` looks like the natural escape hatch but is
**not**: coding-agent credentials are not modeled as definition-level secret refs at all, so it
returns `[]` for a genuinely credentialed definition. Don't rely on it here.)

**`preview_agent_definition` does NOT report availability.** It is merged config — a config view, not
a selection surface.

### If a start is refused
That refusal is **correct behaviour, not a transient fault**. Branch on the machine-readable code in
`error.detail`: `coding_agent_credential_unavailable` (the umbrella) or `credential_unusable`. **Do
not branch on `error.code`** — it is the HTTP status repeated as an integer and discriminates
nothing. **Do not retry.** Fix the credential or pick another definition.

## 1. Start

- **`launch`** — the modern path, for a project/environment.
- **`spawn_runtime`** — the raw path. Needs a `controller_name`; get a valid one from
  `list_controllers`. Prefer `launch`.
- **`create_session`** — a session directly, without a workflow.

`launch` and `spawn_runtime` return a **`task_uuid`**, not a finished runtime.

A sandbox `launch` runs the project's default sandbox profile (its setup script) unless the body
sets `sandbox_profile_uuid` — a profile id from `list_available_sandbox_profiles`, or `"none"`. The
launch reads `setting_up` while the script runs. See `authoring.md` for writing a profile.

## 2. Follow the async part

```
launch → task_uuid → get_task (poll) → get_task_responses (what it produced)
```

**There is no blocking wait tool.** `GET /tasks/{id}/wait` exists in ps-api but has no working
implementation downstream, so it is deliberately not exposed. Poll.

⚠️ A runtime is ready on the **instance**, not the parent runtime record — use `get_launch` /
`list_launch_attempts` and `list_launches`.

## 3. Drive the session

- `send_session_input` — feed it a prompt or command.
- `get_session` — its state. **This is also the completion probe.**
- `stop_session` — ends the session's **agent**; graceful by default, pass `force=true` only after a
  graceful stop has failed. ⚠️ It **does NOT release the sandbox**: the session stays `state=started`
  and the sandbox keeps running.
- `stop_runtime` — releases the sandbox. Pass the session's `runtime_instance_uuid` (from
  `get_session`), **not** a `runtime_uuid`. Every session on that sandbox stops and reaches
  `state=closed`.

## 4. Follow events — the poll loop

`get_session_events` is a **cursor poll**, and its response shape surprises people:

1. Call it with **no cursor**. You get back a **RAW JSON ARRAY**, oldest → newest.
2. Take the **last element's `event_uuid`** and call again with `after=<that uuid>`.
3. Repeat until you get a short or empty page.

**There is NO `next_cursor` and NO done flag in the payload.** Because it is a raw array (not an
object), it also arrives as text only — there is no structured content for this one call, by design:
wrapping it in an envelope would contradict the documented contract.

**Detect completion out of band**: poll `get_session` until `state=closed` (then `closed_at` and
`exit_code` are populated). `state=closed` arrives only when the session's sandbox stops — never
after a graceful `stop_session` alone.

Cursor errors: a malformed uuid → **400**; an unknown or aged-out uuid → **410 Gone**, meaning start
the poll again from the beginning.

## 5. Read the results

- `list_session_artifacts` (optionally `kind=`), then `get_artifact` with the
  `artifact_version_uuid`. Content is **inline UTF-8 text** — not a download link, not binary.
- **`{"artifacts":[]}` is a normal state**, meaning nothing has been collected yet. It is not an
  error and not a reason to retry the run.
- Durable results that outlive a session live at project/workspace scope:
  `list_scoped_artifacts` / `get_scoped_artifact`.
- Artifacts are also addressable as **MCP resources**, if your harness prefers to attach content
  rather than inline it:

  | Resource URI | Ids from |
  |---|---|
  | `psmcp://sessions/{session_uuid}/artifacts/{artifact_version_uuid}` | `list_session_artifacts` |
  | `psmcp://projects/{project_uuid}/artifacts/{artifact_version_uuid}` | `list_scoped_artifacts` (scope=project) |
  | `psmcp://workspaces/{workspace_uuid}/artifacts/{artifact_version_uuid}` | `list_scoped_artifacts` (scope=workspace) |
  | `psmcp://agent-definitions/{agent_definition_uuid}/resolved-files` | `list_agent_definitions` |

  ⚠️ **Your client's resource LISTING will be empty**, and that is not a fault. These are published
  as URI *templates* (`resources/templates/list`), not as an enumerable set (`resources/list`) —
  the readable set is per-tenant and per-caller, so there is nothing tenant-independent to list.
  Reading a concrete URI works normally. If your client only reads `resources/list` it will tell you
  there are no resources; construct the URI from the table above instead.
- An agent's own coding attempts on a project: `list_platformsmith_attempts`, then
  `get_attempt_files` or `get_latest_attempt_files` for "what did the last run change?". Once
  you've opened a PR from an attempt, record it with `set_platformsmith_attempt_pr_url` — the
  write side of the "PR links" `list_platformsmith_attempts` already reads back, and the way to
  close the loop on your own output without a human doing it for you.

## 6. The in-pod tool surface (a SPAWNED session's own MCP tools, not yours)

Everything above is what **you** call from outside. A session you spawn (`session-prompt`,
`create_session`, `launch-playbook`'s workers) is itself granted a **separate** MCP tool surface
inside its own pod, used by the agent running there:

| In-pod tool | Purpose |
|---|---|
| `save_artifact` | Persist an artifact (`name`, `content`, `kind`, `scope`, optional `content_type`). `scope` is `session`, `project` or `workspace`; `kind` is `result` for a session's own output, `recipe` being reserved for platform-written build recipes. Saving the same (`scope`, `name`) again appends a new version. |
| `list_artifacts` | List artifacts the pod can read, at one `scope` (`session` \| `project` \| `workspace`), optionally narrowed by `kind`. Metadata only — pass an `artifact_version_uuid` to the in-pod `get_artifact` for a body. |
| `list_artifacts` / `get_artifact` **scope rule** | `scope` names a **tier, not a target**. There is no `project_uuid` or `workspace_uuid` argument: the platform resolves the scope from the runtime the pod is in, so a session cannot read a sibling project's artifacts. |
| `get_artifact` (in-pod) | Read one artifact's content by `artifact_version_uuid` at the same `scope` you listed it from. Over **1 MiB it is REFUSED, not truncated** — the error names the size, and you never receive a partial body you might mistake for the whole. Unknown, another tenant's, and out-of-scope all return the same not-found. |
| `ps_work_item_attach_artifact` | Attach an already-saved artifact to a work item (`artifact_uuid`, `role`). |
| `ps_work_item_query` | Read work-item state from inside the pod. |
| `ps_work_item_create_child` | Decompose the item the session is attributed to. |
| `ps_work_item_set_state` | Advance the item's execution state. |
| `ps_work_item_claim_ready` | Claim a ready child from a decomposition. |
| `ps_work_item_add_dependency` | Add an ordering edge between items. |
| `ps_work_item_comment` | Annotate the item's activity rail. |
| `ps_signal` | Signal a parked `await-signal` correlation (e.g. a playbook run's completion). |
| `a2a_send` | Send an A2A message from inside the pod. |

⚠️ **`get_artifact` is two different tools.** The one in this table runs **inside a pod** and
reads at a session/project/workspace scope resolved from the runtime. The `get_artifact` in your
own tool list (§ the session-artifact tools above) is a **control-plane** tool you call over
ps-api for a session you name. Same verb, different caller, different reach — if you are calling
it from outside a pod, you want the control-plane one.

**Every `ps_work_item_*` tool needs session attribution to work at all.** A session has no work
item to act against unless one was wired in: `session-prompt` accepts an optional
`work_item_uuid` input (mirroring `launch-playbook`'s field of the same name) — wire it from an
upstream `work-item-create`/`work-item-create-child` node, or the spawned session's
`ps_work_item_attach_artifact` (and siblings) will refuse with "this session is not attributed to
a work item," and no in-pod tool takes the item id as an argument to work around it.

**If a session is already running unattributed, fix it from outside with
`attribute_session_to_work_item`** — the control-plane tool that records an existing session as an
attempt on an item. That is the supported repair for the common case where the work turned out to
matter only after it started, and it is idempotent for the same item. It is a *control-plane* call
(you name the session), so it is not available to the pod itself: the in-pod tools still read
attribution rather than setting it, which is what keeps a pod unable to reassign its own work.

**Finding a legal state before you set one.** `set_work_item_state` validates against the
lifecycle that governs the item's type, and an illegal move answers 409 without telling you which
moves were legal. Read `list_workspace_item_types` for the types a workspace allows (and each
one's lifecycle), then `list_lifecycle_states` and `list_lifecycle_transitions` for where the item
can actually go. Do the same before `create_work_item`: its `item_type_name` defaults to
`"Parent Work"`, and a deployment whose scheme does not define that name refuses the create.

**Prefer the runtime-less `save-artifact` workflow node over an in-pod `save_artifact` call** when
the only reason for the session is to publish text (e.g. summarizing several upstream nodes'
output) — it needs no coding-agent session at all, and in a fan-out the node that produces the
deliverable runs last, on whatever harness quota the parallel branches left behind. A session that
exists only to call `save_artifact` is exactly the one most exposed to that risk.
