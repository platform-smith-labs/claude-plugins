# Authoring: agent definitions, workflows, playbooks, sandbox profiles

Four authorable surfaces — agent definitions, workflows, playbooks and sandbox profiles. Workflows and
playbooks differ in one way that matters a great deal: **workflows have a
draft lifecycle and playbooks do not.**

## Agent definitions

CRUD (`create_`/`get_`/`update_`/`delete_agent_definition`, `reset_agent_definition`) plus **file
bodies** — the actual agent config that lands in the runtime.

`create_agent_definition_file` takes:

| Field | Meaning |
|---|---|
| `concept` | One of: instructions, settings, skills, commands, mcp_servers, hooks, permissions, sandbox, agents, rules, output_styles, plugins |
| `file_path` | Where it lands in the runtime, relative to the working directory: the instructions file is `CLAUDE.md` (Claude Code) or `AGENTS.md` (Codex) at the root (not `.claude/CLAUDE.md`); skills go under `<root><name>/SKILL.md`, root `.claude/skills/` (Claude Code) or `.agents/skills/` / `.codex/skills/` (Codex). `~/` puts a file in the home folder (`~/.codex/config.toml`) |
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

### Two harnesses, one agent

An agent can carry the files of both harnesses, so switching `coding_agent_type` needs no reinstall:

| | Claude Code | Codex |
|---|---|---|
| Instructions | `CLAUDE.md` | `AGENTS.md` (or a link to `CLAUDE.md`) |
| Skills | `.claude/skills/<name>/` | `.agents/skills/<name>/` or `.codex/skills/<name>/` |
| MCP servers | `.mcp.json` | `~/.codex/config.toml`, `[mcp_servers.<name>]` tables |

Keep each skill once and put a **folder link** at the other harness's root. A harness sees only the skills
under its own roots, so **binding a playbook** (or switching a bound agent's `coding_agent_type`) is refused
while a skill that declares `watch_files` is invisible to the agent's harness; the error names the folder
link to add.

`~/.codex/config.toml` must be valid TOML and may not define the `platform-smith` server: the platform adds
its own server next to yours, as it does for `.mcp.json`. It cannot be a link, and it may not set `projects`,
`profile` or `profiles` (another config layer the platform's server could not override).

**Where model calls go is reserved.** No definition file may set the model endpoint, the credential, a command that
produces it, a proxy or certificate, or a telemetry destination: for example `ANTHROPIC_BASE_URL` / `HTTPS_PROXY`
under `env`, `apiKeyHelper`, or Codex `model_provider`, `model_providers` and `otel`. Nor may a secret ref be injected
as one of those names. A save naming one is refused for every caller, admins included, and the refusal is final, so
do not retry or rename the key. To route through a gateway, a company admin sets it on the shared coding-agent
connection (it applies to runtimes launched afterwards).

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

**Two kinds of parent.** `parent.kind` is `"entity"` — `{uuid, name, scope_type, is_archived}`, the item it
was copied from — or `"library"` — `{code, name, version, is_archived}`, a **PlatformSmith Library** item. Every
organization's core playbooks (PlatformSmith Work Item, PlatformSmith Knowledge Base) and default agents
(Default, Default (Codex), Platform Playbooks, PlatformSmith Knowledge Base) are Library copies. For a Library
parent, `version` is the Library version this deployment ships and `parent_revision` the one the copy last
took; when a deployment ships a newer one the copy shows `update_available` and an organization admin
reviews and applies it exactly as below. Nothing updates on its own; `detach_*` stops updates; publish and
apply-to-copies are for parents you own. `is_archived: true` on a Library parent means the item is no
longer shipped: the copy keeps working and no update is offered.

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

## Install from the Library

The Library is the catalogue of ready-made PlatformSmith agents, playbooks and workflows. Installing one gives your
organization an **ordinary, editable copy** — look here before authoring an agent or playbook from scratch.
Connectors are not Library items; they are connected under Integrations.

1. **Browse** — `list_library_items`, optionally with `q` (words over name, summary, guidance and tags),
   `kind` (`agent_definition` | `playbook` | `workflow`), `category` and `harness`. Each item says when to use it
   (`guidance`), which harnesses it supports, which integrations it `needs`, where it can be installed
   (`scopes`) and `installed_at` — your organization's own copies of it, with `update_available` when the
   Library has shipped a newer version.
2. **Look at one** — `get_library_item` with the item's `code`: its files, an instructions preview, the
   secret refs it declares (names only) and, for a playbook, the agent that comes with it.
3. **Plan** — `plan_library_install` with `{kind, workspace_uuid?, project_uuid?, harness?}` — `kind` is the
   item's kind as the list shows it; always send it (a mismatch is `kind_mismatch`). No uuid means the
   **organization** (organization admins only); `workspace_uuid` installs into that workspace; `project_uuid`
   installs an agent into a project (playbooks have no project scope). It writes nothing and returns what
   would be created or reused (a playbook brings its agent), any `conflicts` (a name already used, the item
   already installed there), the resolved `harness`, the **readiness**, and a `plan_hash`. Installing needs the
   right to add agent definitions — and, for a playbook, playbooks — at the target; a custom role missing one
   gets a 403.
   An item whose `action` is `blocked` is named by a conflict (its `uuid` is the existing row) and
   `can_install` is false: tell the person why and do not install.
4. **Show the person** the plan and the readiness before anything is created: what will be created or
   reused and where, which harness, any conflict, and every readiness item that is `missing` with its fix.
5. **Install** — `install_library_item` with the same kind, target and harness plus the plan's `plan_hash`. It is
   all-or-nothing. A changed target or listing since the plan is a 409 (`plan_stale`) — plan again and show it
   again. The body is `{kind, workspace_uuid?, project_uuid?, harness?, mode?, plan_hash}` (`kind` and `plan_hash`
   required, the rest the SAME as the plan); there is no company argument.
6. **Act on readiness** — an install goes ahead even when the item is not ready, and the result repeats the
   readiness. Tell the person each missing piece: a coding-agent credential for the harness, a secret the
   agent declares (add it as a secret — never paste a value into a file), or an integration the item needs
   (connect it under Integrations). Readiness is a snapshot: "ready" is not a promise the credential works.

**Harness.** An agent runs under Claude Code or Codex. Pass `harness` to choose; omit it for the item's
default (a playbook that reuses an agent copy already there keeps that copy's harness). If the item does not
support the harness you asked for, the refusal (422) names the harnesses it does support — offer one of those.
The files install either way, so switching later needs no reinstall.

**Updates.** An installed copy records the Library item as its parent. When the Library ships a newer
version the copy shows `update_available`; review and apply it on the copy with the usual update tools
(see Copies, history and updates above) — an unedited copy takes it whole.

## Library workflows (use as-is or fork)

Use a playbook when an agent should work the task out in a conversation. Use a workflow when the steps are
fixed and should run the same way every time.

A Library workflow (`kind: workflow`) installs the same way — plan, show the person, install — with three
differences:

- **Mode.** Pass `mode` to plan and install: `use_as_is` (the default) makes a **read-only** copy that takes
  each Library update whole; `fork` makes an editable copy whose updates you compare and apply by hand.
  Workflows can't be customised in place: to edit a use-as-is copy, ask the person and call
  `fork_workflow_definition` — a fork can't go back to use-as-is. `mode` is part of the `plan_hash`.
- **What it needs.** The item's `needs` names the agents it uses (installed or reused with it), the step
  types (task types) its steps use, the hosts its steps connect to, and the secrets it declares (names only).
  A step type this deployment does not have is a blocking readiness item: "Uses a step type that isn't
  available here: <type>. This workflow can't be installed." A name already used at the target is a
  `name_taken` conflict.
- **Suggested triggers.** Installed off. You turn each one on. Installing at a workspace or project creates
  each suggested trigger as an automation with `enabled: false` (origin `library:<code>@<version>`);
  installing at the organization creates none — set one up later in a workspace, also off. Nothing turns one
  on for the person — not install, not applying an update. When they ask, go through what it would start and
  switch it on with `enable_automation` (in the app, a person turns it on in Automations).

The agents are installed first, then the workflow copy. If the second step fails
(`workflow_service_unavailable`, `error.details.completed: ["dependencies"]`) the agents are kept: plan again
and install again — they are reused. Updates to a Library workflow copy are read with
`get_workflow_definition_update` and applied with `apply_workflow_definition_update` (Workflow copies,
history and updates below).

## Your org: publish, copy, update

Your organization has its own library of agents, playbooks and sandbox profiles, and its items reach
workspaces and projects by **copy**. The order matters:

1. **Browse your organization's items** — `list_org_library_items`, then `get_org_library_item` for one.
   (This is your own organization's library, not the PlatformSmith Library above.)
2. **Publish first.** `publish_agent_definition`, `publish_playbook` or `publish_sandbox_profile`. A copy
   takes the newest **published** version, so copying an item that was never published is refused with
   **409 `not_published`** — publish it, then copy again.
3. **Copy.** `copy_agent_definition` (to a workspace or a project), `copy_playbook` (organization →
   workspace) or `clone_sandbox_profile`. The copy is complete and editable; the original keeps changing
   independently.
4. **Update, per file.** When the original publishes something newer the copy shows `update_available`.
   `get_agent_definition_update` (or `get_playbook_update`, `get_sandbox_profile_update`) lists each item;
   show the person, then `apply_*_update`. An edited copy needs `"take"` or `"keep"` for every item both
   sides changed — decided file by file, so a copy can take the parent's version of one file and keep its
   own of another.
5. **Make copies update-only (optional, ask first).** `set_agent_definition_copies_read_only`,
   `set_playbook_copies_read_only` or `set_sandbox_profile_copies_read_only`. A read-only copy refuses
   **every** setting edit — rename, description, coding agent type, the mandatory flag, a playbook's
   agent binding, a profile's default — not just file edits, with **409 `read_only_copy`** naming the
   parent. Applying an update, deleting and archiving stay allowed. Tell the person which copy is locked
   and by whom; do not retry the edit. A copy of an **archived** parent reads `read_only: false` and is
   editable again.
6. **See where an item went.** `list_agent_definition_copies`, `list_playbook_copies` and
   `list_sandbox_profile_copies` answer "who has copied this?". They show **linked** copies only —
   detached copies are excluded, as are archived ones. They are for organization admins (the same right
   that applies updates to copies): a caller without it is **refused**, and a uuid that is not yours reads
   as not found.

Refusals carry a machine reason in `detail` and structured context in `details` (for example the
blocking playbooks or runs). Branch on `detail` and show `details` to the person.

Removing things: `delete_agent_definition` is refused with **409 `bound_to_playbooks`** while a live
playbook binds the agent — unbind it with `update_playbook` (set `agent_definition_uuid` to null) or
remove the playbook first. Removing a playbook is in `running-work.md` (Removing a playbook).

### Your org's workflows

A company-level workflow with at least one published version is one of your organization's items:
`list_org_library_items` (`kind=workflow`, also listed unfiltered) and `get_org_library_item` with
`kind: workflow` (its published versions and the agents its steps use). Copy one into a workspace or project
with `clone_workflow_definition`, body `{target_workspace_uuid | target_project_uuid, mode?}` — exactly one
target; `mode` defaults to `fork` here (`use_as_is` for a read-only copy that takes updates whole). The copy
starts from the newest **published** version (`not_published` when there is none). The agents it uses are
your organization's own and are not copied: `warnings` names any the target cannot see. Publishing a newer
version (`publish_workflow_definition`) offers it to every copy. If the list carries
`unavailable_kinds: ["workflow"]`, workflows can't be shown right now — try again in a moment; it does not
mean there are none.

## Export, import and bulk edits

**Export** — `export_agent_definition_bundle` (and `export_sandbox_profile_bundle`, `export_playbook_bundle`)
returns a portable bundle: `{manifest, files:[{path, content}]}`. The manifest (`ps-bundle.json`,
`schema_version: 1`) carries the settings, each file's concept, link target and executable flag, and the
secret refs **by name only**. A playbook bundle nests its bound agent under `agent/`. Components from your organization travel by
`org_component_uuid` (`source: org`, `library_code: null`, plus `detached: true` when the source was unpublished);
PlatformSmith ones travel by `library_code`. Exporting writes nothing.

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
   (`change: imported`); restore the version before it to undo. Importing re-links an organization component
   **only** by its uuid inside the same organization (published, visible to the agent, that version present);
   otherwise it is imported **detached** with the warning `component_listing_unknown` ("names an org component
   that is not available here" — identical for unknown and foreign ones). Show the person that warning. An
   entry with both `library_code` and `org_component_uuid` is `component_invalid`. Restoring or copying a
   version whose organization source no longer resolves also yields a detached component, never an error.

Credential and history files (`.credentials.json`, `history.jsonl`, `.env`, `.ssh/`, `.git/`, private
keys, session transcripts) are never imported — a bundle holding one is refused whole. Files are text only.

**Bulk edits** — `write_agent_definition_files` takes `{upsert:[{file_path, content, …}], delete:[…],
base_version?}` and writes them all as **one version**; rules are checked against the resulting file set,
so a link and its target can arrive together. Prefer it to many single-file calls.

**Duplicate vs copy** — `duplicate_agent_definition` makes a **standalone** agent (no parent, never offered
updates) anywhere you may create agents, including the same workspace under a new name or the
organization. Use `copy_agent_definition` when the new agent should keep taking the source's updates.

### Build an org agent end to end

An organization-level agent that conducts a playbook, from nothing to a finished run. Writes need an
organization admin (403 otherwise). Example names are invented; use the person's own.

1. `create_scoped_agent_definition` with `scope: "company"` (no `scope_id`), e.g.
   `{name: "release-notes-agent", coding_agent_type: "claude_code"}`.
2. `create_scoped_agent_definition_file` for `CLAUDE.md` (`concept: instructions`) and any skills.
   Fix one with `update_scoped_agent_definition_file`, remove one with
   `delete_scoped_agent_definition_file`; `list_agent_definition_files` confirms the set before you go on. To read
   ONE file back use `get_scoped_agent_definition_file` (`agent_definition_uuid`, `file_uuid`) —
   `get_agent_definition_file` demands a `project_uuid` and does not serve an organization agent.
3. `create_playbook` **without** `workspace_uuid` (organization-level) and with `agent_definition_uuid` set to
   the agent from step 1 — or create it first and bind later with `update_playbook`
   (`agent_definition_uuid`; null unbinds). A playbook with no agent cannot run.
4. `start_playbook_run` with body `{"playbook_uuid": "…", "workspace_uuid": "…"}` (the workspace to run in), then follow `get_playbook_run` until it is
   terminal. Read `failure_reason` on a failed run; do not poll faster than the person needs.
5. If the run is refused with `playbook_archived`, the playbook was removed: tell the person, do not retry.
   If removing something is refused with `active_runs`, show `error.details.runs` and ask before
   `cancel_playbook_run`.

### Write a workflow from scratch

1. `get_workflow_task_catalog` for the live node menu; it outranks any list written here.
2. Draft `conductor_json` and call `validate_workflow_definition` (with `execution_context`) until `ok` is true.
3. Read "The plumbing validation does NOT check" below and check each item by hand; validation passing is not
   the same as the workflow working.
4. For an **organization (company) workflow** `create_scoped_workflow_definition` with `scope: "company"` (no
   `scope_id`); `create_workflow_definition` is project-only and demands `project_uuid`. Set
   `execution_context` now, then `publish_workflow_definition`.
5. `run_workflow_in_workspace`, then `get_workflow_execution` for the run and its steps;
   `list_workflow_executions` finds earlier runs. Ask the person before a run that acts on their systems.

The full rules (human gates, secrets, stopping a stuck run) are in "Workflow definitions — validate,
publish, run" below; this recipe only fixes the order.

### Publish to Your org

1. `publish_agent_definition`, `publish_playbook`, `publish_sandbox_profile` or `publish_workflow_definition`.
   Copies take the newest **published** version; an unpublished item is refused `not_published`.
2. Confirm it is visible: `list_org_library_items` (add `q` for a name, or `kind` for a kind) and
   `get_org_library_item`.
3. Copy it where it is needed: `copy_agent_definition`, `copy_playbook`, `clone_sandbox_profile` or
   `clone_workflow_definition` (body `{target_workspace_uuid | target_project_uuid, mode?}`).
4. When you publish something newer the copies show `update_available`; take it per file as described in
   "Your org: publish, copy, update". Components are published and attached by the steps in "Your org's
   components".

## Workflow definitions — validate, publish, run

1. **`get_workflow_task_catalog`** — the LIVE node menu and per-node input schemas. **It outranks
   anything written down here.**
2. Assemble `conductor_json`: an object with a top-level string `name` and a `tasks` array.
3. **`validate_workflow_definition`** before creating — structural problems surface as
   `{ok, errors[]}` instead of a bare 422 at publish. It also accepts `execution_context` and
   returns a `warnings[]` array (never blocking) when it's missing — see step 4.
4. `create_workflow_definition` (project) or `create_scoped_workflow_definition` (company) → `publish_workflow_definition` → `run_workflow`. **Set
   `execution_context` (`"workspace"` or `"project"`) at CREATE time** — both create tools accept
   it — rather than discovering it's required only when publish 422s after the graph is already
   persisted. A run always needs a workspace: use **`run_workflow_in_workspace`**, which always
   carries one (`run_workflow` is refused with 422 when none resolves).
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
- A run always needs a workspace: start it with `run_workflow_in_workspace`, which always carries
  one. `company_uuid` and `user_uuid` are added for you.
- A **`FORK_JOIN_DYNAMIC` must be immediately followed by a `JOIN`**, or the run fails.
- **Dynamically-forked tasks are not covered by publish-time tenant injection** — publishing stamps
  `_ps.company_uuid` into tasks it can *see*, and a dynamic fork's sub-tasks are born at run time.
  This is why a hand-rolled fan-out fails however correct it looks. Use `resolve-projects` with a
  `_ps.fork` spec, which emits `fork_tasks` + `fork_inputs` with context already stamped.
- Do **not** feed `resolve-projects`' `projects` output to `dynamicForkTasksInputParamName` — that is
  project *data*, not a task array. Use `fork_tasks` + `fork_inputs`.
- A fan-out over N repos where only k have work is **normal**: `git-open-pr` classifies "no commits
  between" as a SKIP with a reason, not an error.

## Workflow copies, history and updates

A workflow joins the copy model with the same tools as an agent or playbook
(`list_workflow_definition_versions`, `compare_workflow_definition_versions`,
`restore_workflow_definition_version`, `get_workflow_definition_update`, `apply_workflow_definition_update`,
`list_workflow_definition_copies`, `apply_workflow_definition_to_copies`,
`set_workflow_definition_copies_read_only`, `detach_workflow_definition`) and two differences:

- **No per-item merge.** Apply replaces the whole graph with the parent's version and keeps the copy's
  settings (name, description, category, labels). On an edited fork, ask first: it replaces the fork's own
  graph edits, and you can restore it from History. A use-as-is copy takes the update with no question.
- **One Publish.** Restoring makes a new version from an old one. It isn't live until you publish it. The
  same holds for Apply: the person publishes with `publish_workflow_definition` after review.

`get_workflow_definition` shows `copy_mode` (`use_as_is` | `fork`), `read_only_reason` (`use_as_is` |
`parent_locked`) and `update_available` (`{version}` or null). A parent's read-only lock beats Fork. Runs
already started keep the version they started with. Branch on `error.detail` for every refusal
(`reference/troubleshooting.md`).

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

⚠️ **The difference that will bite you**: playbooks have **no validate and no draft**.
What you write is live immediately for every subsequent `start_playbook_run`. (`delete_playbook` archives one that has no active runs.)
Every save is recorded, so `restore_playbook_version` undoes a bad edit. (`publish_playbook` is not a
draft gate: it only offers the current version to the playbook's copies.)

`update_playbook` is a **partial** update despite the PUT verb: omitted fields are **preserved**
(verified live). You do not need to read-modify-write the whole object to change one field.

So:

1. Name it deliberately — it is live on write (`delete_playbook` archives it later, but only with no active runs).
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

## Components

A **component** is a skill, MCP server, hook, subagent, rule, script or connector entry that is tracked as one
unit inside ONE agent definition. Components are **attached**, never installed: browse them with
`list_library_items` (`kind=component`, optionally `component_kind`), then:

1. `plan_attach_agent_definition_component` (`library_code`) — a dry run. **Show the person** the plan: the
   files and entries it adds, what it owns, what it executes and connects to, every clash, the secret
   references (names only) and the readiness. `can_apply: false` means stop and say why.
2. The person decides: each clash is `take` (the component's version) or `keep` (the agent's own); each
   secret name is `reuse` or `cancel`; `executes` / `connects` are acknowledged only after they have read
   them. Never fill these in for them.
3. `attach_agent_definition_component` with body `{library_code | org_component_uuid (exactly one, as planned),
   plan_hash, base_version, decisions?, secret_ref_decisions?, acknowledge?}` — `plan_hash` from the plan and
   `base_version` the agent's current version (both are only learnable from a 400 otherwise).
   One new version of the agent is recorded.

Later: `list_agent_definition_components` shows `update_available`; `get_agent_definition_component_update`
shows the offer, `compare_agent_definition_component_update` one item side by side, and
`apply_agent_definition_component_update` applies it with the person's `take` / `keep` decisions.
`apply_agent_definition_unedited_updates` applies everything nobody edited and lists what it skipped (`edited`, `invalid`, or `acknowledgement_required` when a
component update adds a new executes/connects capability — apply that one by one with `acknowledge`). `base_version` is required.
`detach_agent_definition_component` stops update checks and keeps all content; `remove_agent_definition_component`
deletes the component's files, entries and its own secret references. Ask the person first for both.

A `harness_not_supported` (422) refusal is final: name the harness and suggest another listing. A
`plan_stale` or `stale_base_version` (409) means re-plan and show the person again.

An `entry_owned` clash is fixed by removing or detaching the component that owns the entry, never by taking
its key. Hooks and subagents are Claude Code only: for a Codex agent the plan says so and
`harness_not_supported` is the answer. Reading a component's secret references shows names only; whether to
reuse an existing secret or cancel is the person's call. The table in `reference/troubleshooting.md`
("A component would not attach, update or remove") maps each refusal to its next step.

### Your org's components

Your organization can publish its **own** components, next to the PlatformSmith ones. They are attached
the same way (plan, show, attach) but by `org_component_uuid` instead of `library_code`; give exactly one.

- **Browse**: `list_org_library_items` with `kind=component` (optional `component_kind`; add `workspace_uuid`
  and/or `project_uuid` to see what an agent there can see — with neither you get organization-wide
  listings only). **Look**: `get_org_library_item` (`kind=component`) or `get_org_component`;
  `get_org_component_version` reads one past version (an integer).
- **What can this agent add?** `list_available_agent_definition_components` lists your organization's and
  PlatformSmith components for the agent's coding agent (`source` is `org` or `platform`;
  `hidden_for_harness` counts the ones left out). It is not `list_agent_definition_components`, which lists
  what is already attached.
- **Attach**: `plan_attach_agent_definition_component` with `org_component_uuid`. **Show the person**
  "Published by <name> in <scope>" (no name means a former member) with the files, entries, what it runs
  and connects to. PlatformSmith does not review components your organization publishes: only basic checks
  run, and reviewing what a component runs and connects to is your organization's responsibility. Then
  `attach_agent_definition_component` as above. `item_not_found` means the agent cannot see that
  component; `org_component_unpublished` and a 409 `unpublished` are final. An organization plan carries
  `source: "org"`, `org_component_uuid` and `publisher` with `library_code: null`; an attached component whose source
  was unpublished reads detached (`detached_reason: source_unpublished`); one whose listing cannot be resolved here (another company, deleted, not visible) reads detached with `detached_reason: source_unavailable`. A detached organization row keeps `source: "org"` and its `code` (the listing code; `org_component_uuid` may be null), and its update offer, compare and apply
  return 409 `component_detached`, and the bulk update skips it. An organization component otherwise updates
  exactly like a PlatformSmith one (same offer, compare, take/keep decisions, apply).
- **Author from an agent**: `draft_org_component_from_agent_definition` (body `{component_kind, name, code?,
  paths?, entries? [{file_path, entry_key}], secret_refs?}`; writes nothing; send the files,
  config entries and secret NAMES to include — at least one file or entry). Review the draft with the
  person, then `create_org_component` with `dry_run: true` and fix every problem it lists (read each
  `message` against its `path` / `entry_key`), show the person the final files, entries, what it
  executes and connects to, **ask**, then create it for real with `from_agent`. `from_agent.base_version` is the
  agent's version; leave tracking on to keep the agent following the component. To author from scratch,
  call `create_org_component` without `from_agent`.
- **Where it is published**: neither `workspace_uuid` nor `project_uuid` is organization-wide; one of them
  is that workspace or project; both is a 400. Everyone in that scope can see it. Organization owners and
  admins publish at organization level; workspace admins and editors publish in their workspace and its
  projects.
- **New version**: `publish_org_component_version` with `base_version` = the component's latest version
  (from `get_org_component`), `dry_run` first. Versions never change; agents tracking it are offered the
  update through the usual `get_agent_definition_component_update` flow.
- **Unpublish**: `unpublish_org_component` with `confirm: true`, only after the person agrees. Agents using
  it keep their files but stop getting updates (`detached_agents` lists them); it can no longer be added,
  and there is no republish. Unpublishing needs the delete permission, so editors cannot do it.
- **Rules**: the code is yours alone and reserved forever, even after unpublish (`code_exists` /
  `code_reserved` mean pick another). A component holds secret **names** only, never a value. Size limits:
  at most 100 files, 1 MiB per file, 2 MiB of content and 100 entries per version.

## Suggest components for a goal

When the person states a goal ("I want the agent to review pull requests"), search before you write anything.
You do the ranking; the platform only filters.

1. Say the goal in one sentence and pull out two or three plain words.
2. `list_org_library_items` with `q=<words>` and `kind=component`, then `list_library_items` with `q=<words>`,
   `kind=component` and optionally `component_kind`. Look at your organization's items first, then the Library.
3. `q` is words: case-insensitive, every word must match, at most 200 characters. An empty result is
   `items: []`, not an error; try fewer or different words before giving up.
4. Read the summaries only. Pick at most three, and tell the person which and why.
5. **Plan before attaching:** `plan_attach_agent_definition_component` and show the plan (see Components).
   Attach only after the person approves.

## Start from the Library

To begin a new agent or playbook from a ready-made one rather than a blank file:

1. `list_library_items` (`q`, `kind`) to browse, then `get_library_item` with the item's `code` for its files and guidance.
2. `plan_library_install` with the target (`workspace_uuid` and/or `project_uuid`, or neither for the
   organization). Show the person the plan and any blockers.
3. `install_library_item` with the same target and harness the plan used.
4. The install is an ordinary editable copy: `list_agent_definition_files` to see it, then edit with the scoped
   file tools in "Build an org agent end to end".
   A Library item's `CLAUDE.md` is a **link row** (`link_target` `AGENTS.md`) and refuses content with a 400:
   edit `AGENTS.md`. An install cannot rename — the copy keeps the Library name; rename it afterwards with
   `update_agent_definition` (a full PUT: send the whole body with the new `name`).
