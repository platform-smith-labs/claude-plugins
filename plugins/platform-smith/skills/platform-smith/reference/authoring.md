# Authoring: agent definitions, workflows, playbooks, sandbox profiles

Three authorable surfaces. They differ in one way that matters a great deal: **workflows have a
draft lifecycle and playbooks do not.**

## Agent definitions

CRUD (`create_`/`get_`/`update_`/`delete_agent_definition`, `reset_agent_definition`) plus **file
bodies** — the actual agent config that lands in the runtime.

`create_agent_definition_file` takes:

| Field | Meaning |
|---|---|
| `concept` | One of: instructions, settings, skills, commands, mcp_servers, hooks, permissions, sandbox, agents, rules, output_styles, plugins |
| `file_path` | Where it lands in the runtime (e.g. `.claude/CLAUDE.md`) |
| `merge_strategy` | `append` \| `replace` \| `deep_merge` — how it combines with the same path inherited from an outer scope |
| `content` | The file's text |

**Configuration merges inner-over-outer**: environment beats project beats workspace. So the file you
wrote is not necessarily the file that runs — read **`get_agent_definition_resolved_files`** for the
effective merged set, which is what actually lands in the container.

Secrets: attach **refs**, never values (`add_agent_definition_secret_ref`); check they resolve with
`get_agent_definition_secret_refs_status`.

**`create_agent_definition_file` only reaches a project-scoped definition** — it takes
`project_uuid` as a path segment, so a company- or workspace-scoped definition 404s ("agent
definition not found"), which reads as a tenancy problem but is a routing one. Use
**`create_scoped_agent_definition_file`** (no scope/scope_id needed — the target definition's own
scope is already fixed) for anything above project scope.

## Workflow definitions — validate, publish, run

1. **`get_workflow_task_catalog`** — the LIVE node menu and per-node input schemas. **It outranks
   anything written down here.**
2. Assemble `conductor_json`: an object with a top-level string `name` and a `tasks` array.
3. **`validate_workflow_definition`** before creating — structural problems surface as
   `{ok, errors[]}` instead of a bare 422 at publish. It also accepts `execution_context` and
   returns a `warnings[]` array (never blocking) when it's missing — see step 4.
4. `create_workflow_definition` → `publish_workflow_definition` → `run_workflow`. **Set
   `execution_context` (`"workspace"` or `"project"`) at CREATE time** — both create tools accept
   it — rather than discovering it's required only when publish 422s after the graph is already
   persisted. Use **`run_workflow_in_workspace`** when the definition is company-scoped: it owns
   no workspace and must be told which one to run in.
5. Human gates: an `await-signal` node with `_ps.shape:"approval"` parks and surfaces in
   `get_workflow_inbox`; decide it with `create_workflow_approval` using `approved` / `rejected`
   — **past tense, exactly those two values**.
6. A run that will never finish (a stalled turn with no deadline reached, a fork branch that
   quota-refused with nothing downstream to notice) can be stopped with
   **`terminate_workflow_execution(execution_id, reason)`**. A live run stops. A run that has
   already finished answers **409** with `error.detail` = `execution_already_terminal` and
   `error.fields.status` giving its final status — treat that as "already finished", not a failure, and read
   `get_workflow_execution` for the outcome. A missing run, or another company's, answers **404**.
   So it is safe to call on anything `list_workflow_executions` shows as suspiciously
   long-`RUNNING`, even if the run finishes first.

Secrets: attach **refs**, never values (`add_workflow_definition_secret_ref` — the exact
counterpart of the agent-definition verb above); check they resolve with
`get_workflow_definition_secret_refs_status`. There is no detach for either family — that's
config surgery a human does knowingly.

### The plumbing validation does NOT check

`validate_workflow_definition` checks **structure only**. These fail at **run** time instead, and are
the usual reason a graph that validated cleanly still dies:

- **Worker nodes read all inputs — including the "user" ones — from an `_ps` object** on the task.
  Put `content`, `session`, `only_with_repo` etc. *inside* `_ps`, not at the task top level.
  (Conductor SYSTEM tasks — `JSON_JQ_TRANSFORM`, `FORK_JOIN_DYNAMIC`, `JOIN`, `HTTP`, `WAIT`,
  `INLINE`, `SET_VARIABLE` — take **no** `_ps`.)
- **Tenant context is not inherited.** Every worker node must reference it explicitly:
  `company_uuid: ${workflow.input._ps.company_uuid}` (likewise `workspace_uuid`, `user_uuid`). A task
  whose `_ps` lacks `company_uuid` fails with `_ps missing or invalid company_uuid`.
- Supply the workspace at **run** time: `run_workflow` with input `{_ps:{workspace_uuid:"…"}}`.
  `company_uuid` and `user_uuid` are added for you.
- A **`FORK_JOIN_DYNAMIC` must be immediately followed by a `JOIN`**, or the run fails.
- **Dynamically-forked tasks are not covered by publish-time tenant injection** — publishing stamps
  `_ps.company_uuid` into tasks it can *see*, and a dynamic fork's sub-tasks are born at run time.
  This is why a hand-rolled fan-out fails however correct it looks. Use `resolve-projects` with a
  `_ps.fork` spec, which emits `fork_tasks` + `fork_inputs` with context already stamped.
- Do **not** feed `resolve-projects`' `projects` output to `dynamicForkTasksInputParamName` — that is
  project *data*, not a task array. Use `fork_tasks` + `fork_inputs`.
- A fan-out over N repos where only k have work is **normal**: `git-open-pr` classifies "no commits
  between" as a SKIP with a reason, not an error.

## Playbooks — authorable, but LIVE ON WRITE

`create_playbook` and `update_playbook` author a playbook definition.

`create_playbook` takes `{name, workspace_uuid, description?, tracks_work_items?}` — the first two
are required and ps-api 400s with the offending field named.

⚠️ **The difference that will bite you**: playbooks have **no validate, no publish and no delete**.
There is no draft state, so what you write is live immediately for every subsequent
`start_playbook_run` — and permanent, because nothing removes it.

`update_playbook` is a **partial** update despite the PUT verb: omitted fields are **preserved**
(verified live). You do not need to read-modify-write the whole object to change one field.

So:

1. Name it deliberately — you cannot delete it later.
2. Write it right the first time; there is no validate step to catch you.
3. Run with `start_playbook_run`; follow with `get_playbook_run`; history via `list_playbook_runs` /
   `list_workspace_playbook_runs`.

## Sandbox profiles — the setup a project's sandboxes need

A sandbox profile is a bash setup script that runs in a new sandbox after the repository is cloned
and before the coding agent starts (passwordless sudo). Use one when a project needs a toolchain,
a service or a dependency install the base sandbox image lacks.

1. **Read the project first** — its manifests, `Dockerfile`, CI config — so the script installs
   what the project actually uses.
2. **See what already applies**: `list_available_sandbox_profiles` (scope=project) returns every
   profile the project can use and `resolved`, the one a launch with no choice would run.
3. **Write it**: `create_sandbox_profile` at the narrowest scope that fits (usually the project).
   To specialise an organization profile, `clone_sandbox_profile` into the project and
   `update_sandbox_profile` the copy — the source is unaffected.
4. **Secrets are references**, never script text: `add_sandbox_profile_secret_ref` with the
   secret's name, then `get_sandbox_profile_secret_refs_status` with the project_uuid you will launch.
   Secrets reach only the script, masked in its output.
5. **Hand things to the agent** by appending `NAME=value` lines to `$PS_ENV` and directories to
   `$PS_PATH`; the agent's commands see them, not the secrets.
6. **Try it before making it the default**: `launch` with `sandbox_profile_uuid` set to the new
   profile and read the launch timeline (`setup_script_*` events). Then, if every launch should run
   it, `set_sandbox_profile_default`.

⚠️ **Live on write, like playbooks.** A profile resolves for every later launch as soon as it is the
only one in its scope or the marked default; `update_sandbox_profile` changes what later launches
run. `on_failure: continue` (the default) lets the agent start after a failed script, with a
warning; `fail_launch` stops the launch. `archive_sandbox_profile` takes a profile out of
resolution and `restore_sandbox_profile` brings it back.

## Composing a playbook and a workflow

This is the pattern the two surfaces exist for together: the **playbook** carries the agent-facing
procedure, the **workflow** orchestrates around it (fan-out, approvals, notifications, PRs).

1. Author the playbook first — it is live on write, so get it settled before anything references it.
2. Reference it from a workflow via the **`launch-playbook`** node. Check
   `get_workflow_task_catalog` for that node's exact input schema rather than assuming it.
3. Validate and publish the **workflow** (the playbook needs neither).
4. Run the workflow; follow the playbook runs it starts with `get_playbook_run`.

## Scoped authoring

Several tools take `scope` (`project` | `workspace` | `company`) plus `scope_id`. For
**`scope=company`, OMIT `scope_id`** — the company comes from your identity, and the path is
`/company/…`, not a UUID route. Scoped tools: `create_secret`, `get_workflow_inbox`, the
`list_`/`create_scoped_` definition tools, and `list_sandbox_profiles` / `create_sandbox_profile`
(`list_available_sandbox_profiles` takes `project` or `workspace` only).
