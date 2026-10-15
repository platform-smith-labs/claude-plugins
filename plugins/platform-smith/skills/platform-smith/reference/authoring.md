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
| `file_path` | Where it lands in the runtime, relative to the working directory: the instructions file is `CLAUDE.md` at the root (not `.claude/CLAUDE.md`); skills go under `.claude/skills/<name>/SKILL.md` |
| `content` | The file's text (empty for a link) |
| `link_target` | Optional. Makes the row a **link** to another path in the same definition — see *Links* below |
| `is_executable` | Optional. `true` writes the file with mode `0750` instead of `0640` (scripts, hooks) |

**What a definition holds is exactly what runs.** Nothing is merged in from an organization or
workspace definition at launch, so `list_agent_definition_files` is the file set the runtime gets.
Copies, history and updates are below.

### Links

A file with `link_target` is a symlink to another path **in the same definition**, and holds no content.
Use one to give two harnesses the same file without keeping two copies in step:

- `CLAUDE.md` → `AGENTS.md` (a file link), or
- `.agents/skills/review` → `.claude/skills/review` (a folder link — name the folder, no trailing slash).

The target must exist, must not itself be a link or sit inside a linked folder, and must be in the same
root (both relative to the working directory, or both under `~/`). `tools.json`, the settings files and
`.mcp.json` cannot be links. Deleting or renaming a file that a link points at is refused, naming the
link. The sandbox gets a relative symlink.

Secrets: attach **refs**, never values (`add_agent_definition_secret_ref`); check they resolve with
`get_agent_definition_secret_refs_status`.

**The `*_agent_definition_file` tools only reach a project-scoped definition** — they take
`project_uuid` as a path segment, so a company- or workspace-scoped definition 404s ("agent
definition not found"), which reads as a tenancy problem but is a routing one. Use the
**`*_scoped_agent_definition_file`** family — `create_`, `get_`, `update_`, `delete_` — for a definition
at any scope (no scope/scope_id needed; the definition's own scope is already fixed).
`update_scoped_agent_definition_file` is partial: send only the fields that change.

## Copies, history and updates (agent definitions, playbooks, sandbox profiles)

An agent definition, playbook or sandbox profile at a workspace or project is **its own complete copy**.
A copy records its **parent** (the item it was copied from, one level out) and the parent version it last
took. Every read carries `version`, `parent`, `parent_revision`, `edited`, `update_available`,
`detached_at`, `copies_read_only` and `read_only`.

**Make a copy**: `copy_agent_definition` (into a workspace or a project), `copy_playbook` (organization →
workspace; playbooks have no project scope), or `clone_sandbox_profile`.

**History.** Every save that changes something records a version — `list_agent_definition_versions`,
`get_agent_definition_version`, `compare_agent_definition_versions`. `restore_agent_definition_version`
makes an old version current as a **new** version, so nothing is ever lost and a restore can itself be
undone. Deleting a file is a version too.

**Updates are applied by a person, never pushed:**

1. The parent's owner saves, then **publishes**: `publish_agent_definition`. **Only published versions
   count as updates** — saving is never gated and an unpublished edit already runs for the parent itself.
2. A copy whose parent published something newer shows `update_available: {version}`.
3. On the copy, `get_agent_definition_update` shows each item — a file (by path), the secret refs, the
   settings — with a status: `theirs_changed` (only the parent changed; taken by default),
   `mine_changed` (only the copy changed; kept by default), `both_changed` (**must be decided**),
   `added`, `removed`, `unchanged`.
4. Show the person, then `apply_agent_definition_update` with `{version, decisions}`. An **unedited** copy
   (`edited: false`) takes the update whole — send `{version}` alone. An **edited** copy needs
   `"take"` or `"keep"` for every `both_changed` item.

**One level at a time.** A project copy's parent is the workspace copy, so an organization change reaches
the project only after the workspace copy has applied it and published that version.

**On a parent** (ask the person first): `apply_agent_definition_to_copies` updates every unedited copy
that is behind and lists the edited ones it skipped — those need step 3–4 on their own.
`set_agent_definition_copies_read_only` makes copies update-only: applying still works, but editing,
restoring or detaching one is refused (409 naming the parent). **Detach**
(`detach_agent_definition`) stops a copy's updates for good; there is no re-attach.

Playbooks and sandbox profiles have the same verbs: `list_playbook_versions`, `publish_playbook`,
`get_playbook_update`, `apply_playbook_update`, `apply_playbook_to_copies`, `detach_playbook`, …, and
`list_sandbox_profile_versions`, `publish_sandbox_profile`, `get_sandbox_profile_update`,
`apply_sandbox_profile_update`, … Their items differ: a sandbox profile has its script, secret refs and
settings; a playbook its conductor instructions and settings (including the bound agent).

In version, compare and update results each item is `{key, kind, …}`: a file item carries `path`,
`content`, `link_target` and `is_executable`; a settings or secret-refs item carries its values in
`fields` (an object). Secret-ref items hold names and usage only — never a secret value.

**Who may write**: a workspace admin edits their own workspace's agents, playbooks and sandbox
profiles (including its projects'); organization-level ones need an organization admin. Anything
else answers **403**; check `get_my_permissions`.

## Export, import and bulk edits

**Export** — `export_agent_definition_bundle` (and `export_sandbox_profile_bundle`, `export_playbook_bundle`)
returns a portable bundle: `{manifest, files:[{path, content}]}`. The manifest (`ps-bundle.json`,
`schema_version: 1`) carries the settings, each file's concept, link target and executable flag, and the
secret refs **by name only**. A playbook bundle nests its bound agent under `agent/`. Exporting writes nothing.

**Import is always plan, then apply:**

1. `plan_agent_definition_import` (into an existing agent) or `plan_agent_definition_import_as_new` (with
   exactly one of `company: true`, `workspace_uuid`, `project_uuid`; `name` to rename). Send the bundle as
   `bundle: {manifest?, files:[{path, content, executable?, link_target?}]}` — exactly what an export returns —
   or a base64 zip in `bundle_zip`. Without a manifest the files are read as an agent's files at their paths
   (a raw `.claude` folder works). Into an existing agent, `mode: "merge"` (default) keeps the files the bundle
   does not mention; `mode: "replace"` removes them and the plan lists each one.
2. **Show the person the plan.** It writes nothing and returns every path's `action` (add, update,
   unchanged, remove, refused), `refused` (fix the bundle and plan again), `secrets_detected` (literal keys,
   by path and line — replace them with `{{secret:NAME}}` placeholders rather than acknowledging them) and
   `review` (hooks, permissions, MCP servers, Codex `config.toml`, executables, the tools file).
3. `apply_agent_definition_import` / `import_agent_definition_as_new` with the **same** bundle and options,
   `plan_hash` from the plan, and `acknowledge` with the `requires_acknowledgement` values the person agreed
   to. A changed target since the plan is a 409 — plan again. One import is **one version**
   (`change: imported`); restore the version before it to undo.

Credential and history files (`.credentials.json`, `history.jsonl`, `.env`, `.ssh/`, `.git/`, private
keys, session transcripts) are never imported — a bundle holding one is refused whole. Files are text only.

**Bulk edits** — `write_agent_definition_files` takes `{upsert:[{file_path, content, …}], delete:[…],
base_version?}` and writes them all as **one version**; rules are checked against the resulting file set,
so a link and its target can arrive together. Prefer it to many single-file calls.

**Duplicate vs copy** — `duplicate_agent_definition` makes a **standalone** agent (no parent, never offered
updates) anywhere you may create agents, including the same workspace under a new name or the
organization. Use `copy_agent_definition` when the new agent should keep taking the source's updates.

### Recipe: an organization-level agent that conducts a playbook

1. `create_scoped_agent_definition` with `scope: "company"` (no `scope_id`), e.g.
   `{name, coding_agent_type: "claude_code"}`.
2. `create_scoped_agent_definition_file` for `CLAUDE.md` (`concept: instructions`) and any skills.
   Fix one with `update_scoped_agent_definition_file`, remove one with
   `delete_scoped_agent_definition_file`; `list_agent_definition_files` shows the set.
3. `create_playbook` **without** `workspace_uuid` (organization-level) and with
   `agent_definition_uuid` set to the agent from step 1 — or bind later with `update_playbook`.
4. `start_playbook_run` with the `workspace_uuid` to run in, then follow `get_playbook_run`.

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
   `list_notifications` (as a row with `response_spec: "approval"`); decide it with
   `create_workflow_approval` using `approved` / `rejected` — **past tense, exactly those two
   values** — or answer that row with `answer_notification` (`choice: "approved"|"rejected"`);
   both reach the same first-wins decision. (An automation's **held run** is a different approval
   row — kind `automation` — answered with `choice: "approve"|"decline"`, or with
   `approve_automation_run` / `decline_automation_run`; see `reference/automations.md`.)
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
`get_workflow_definition_secret_refs_status`. Detach with `remove_agent_definition_secret_ref` /
`remove_workflow_definition_secret_ref` — it only unlinks; the secret survives and can be re-attached.

### Calling an authenticated API: `rest-call`, never `HTTP`

Whenever a call needs a credential, use the **`rest-call`** node, not Conductor's `HTTP` system
task. They look interchangeable and are not:

- **`rest-call` is worker-executed**, so it carries tenant context and resolves a secret at run
  time. `_ps.secret_headers` maps a header **name** to the **name of a secret ref** declared on the
  definition — `{"Authorization": "openrouter_key"}`. The value is resolved inside the worker and
  never reaches the engine. Literal, non-secret headers go in `_ps.headers`.
- **`HTTP` is engine-executed.** It carries no `_ps`, so it *cannot* resolve a credential — and
  whatever you put in its `inputParameters` is **persisted by the engine and visible in run views**.
  A key placed there is exposed to anyone who can read the definition or any run of it. Use `HTTP`
  only for calls that need no credential at all.

Recipe: `add_workflow_definition_secret_ref`, reference its name in `_ps.secret_headers`, then
confirm it resolves with `get_workflow_definition_secret_refs_status` **before** running. Note the
two different failure shapes: `rest-call` **COMPLETES** with `output.ok=false` on a 4xx/5xx (branch
on `ok`), but **FAILS**, sending nothing, when a referenced secret cannot be resolved — which is
also the first thing to check when a `rest-call` node fails naming a secret.

An LLM gateway (OpenRouter, or any OpenAI-compatible endpoint) is just this pattern: `rest-call` to
the gateway's `/chat/completions` with a bearer credential in `_ps.secret_headers`. The `llm` node
stays the ergonomic choice for a plain reasoning step with `response_schema` validation.

### The plumbing validation does NOT check

`validate_workflow_definition` checks **structure only**. These fail at **run** time instead, and are
the usual reason a graph that validated cleanly still dies:

- **Worker nodes read all inputs — including the "user" ones — from an `_ps` object** on the task.
  Put `content`, `session`, `only_with_repo` etc. *inside* `_ps`, not at the task top level.
  (Conductor SYSTEM tasks — `JSON_JQ_TRANSFORM`, `FORK_JOIN_DYNAMIC`, `JOIN`, `HTTP`, `WAIT`,
  `INLINE`, `SET_VARIABLE` — take **no** `_ps`. That is exactly why `HTTP` cannot carry a
  credential; see *Calling an authenticated API* above.)
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

`create_playbook` takes `{name, workspace_uuid?, agent_definition_uuid?, description?,
conductor_instructions?, tracks_work_items?}`. Only `name` is required. **Omit `workspace_uuid`** for an
organization-level playbook (it runs in every workspace; creating one needs an organization admin).

**Bind an agent, or it cannot run.** `agent_definition_uuid` is the agent that conducts each run; a
playbook without one is refused by `start_playbook_run` with **422**. An organization playbook may bind
only an organization-level agent; a workspace playbook binds an organization agent or one of its own
workspace (400 otherwise). `update_playbook` with `agent_definition_uuid: <uuid>` binds, `null` unbinds,
omitted leaves it. `list_company_playbooks` lists the organization-level ones.

⚠️ **The difference that will bite you**: playbooks have **no validate, no draft and no delete**.
What you write is live immediately for every subsequent `start_playbook_run`, and nothing removes it.
Every save is recorded, so `restore_playbook_version` undoes a bad edit. (`publish_playbook` is not a
draft gate: it only offers the current version to the playbook's copies.)

`update_playbook` is a **partial** update despite the PUT verb: omitted fields are **preserved**
(verified live). You do not need to read-modify-write the whole object to change one field.

So:

1. Name it deliberately — you cannot delete it later.
2. Write it right the first time; there is no validate step to catch you (history lets you go back).
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
   `update_sandbox_profile` the copy — the source is unaffected, and the copy is offered the source's
   later **published** versions as updates (see *Copies, history and updates*).
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
`/company/…`, not a UUID route. Scoped tools: `create_secret`, the
`list_`/`create_scoped_` definition tools, and `list_sandbox_profiles` / `create_sandbox_profile`
(`list_available_sandbox_profiles` takes `project` or `workspace` only).
