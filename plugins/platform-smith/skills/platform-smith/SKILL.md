---
name: platform-smith
description: Drive the PlatformSmith control plane over MCP (ps-mcp) — orient in a tenant, launch runtimes and coding-agent sessions, follow them to completion, read results, author agent definitions, workflows and playbooks (and copy them, read their history, and apply their updates), set up automations (pull-request, tracker-comment, Slack, schedule and webhook triggers, comment commands, record-only work-item capture), govern which connector tools agent sessions may call (connectors, guardrails, the tool audit), and diagnose failures. Use whenever the ps-mcp MCP server is connected and the task involves PlatformSmith workspaces, projects, environments, runtimes, sessions, workflows, playbooks, automations, work items or artifacts.
---

# PlatformSmith over MCP

You are connected to **PlatformSmith** through `ps-mcp`, a thin proxy in front of the platform API.
Everything you do runs **as the user who owns the access token**, scoped to that user's company. You
cannot see or touch another company's data — an attempt returns "not found", which is correct
behaviour and not an error to retry.

## The single most important habit

**List, never guess an ID.** Every tool takes UUIDs. The platform is a nested hierarchy and each
level's list is the input to the next:

```
workspace → project → environment → { agent definitions, workflow definitions, playbooks }
```

Start with `list_workspaces`, then `list_projects`, then `list_environments`. A guessed UUID does not
produce a helpful error — it produces a 404 indistinguishable from a permissions problem.

## Read this before you start anything

Starting a runtime or a session costs real resources and, in the case of a coding agent, turns a
model loose on a customer's repository. Two checks come first:

1. **Is the agent definition startable?** Read `harness_credential_configured`. `false` means there
   is **no usable credential** — the start is *not* refused, it succeeds and the agent fails at its
   first turn. **ABSENT means UNKNOWN — not available.** See
   `reference/running-work.md`; this is the single most common way an agent wastes a turn here.
2. **Is the target ready?** `get_readiness` for the workspace/environment/project, and
   `get_workspace_controllers_summary` — a disconnected controller looks exactly like a stuck launch.

## Where to look next

| You want to… | Read |
|---|---|
| Connect a client and authenticate | `reference/connecting.md` |
| Find the right tool by name and see what it costs | `reference/tools.md` (generated) |
| Launch, run a session, follow it, get results | `reference/running-work.md` |
| Check whether a Dockerfile generation validated and find its pull request | `reference/running-work.md` §7 |
| Author agent definitions, workflows, playbooks, sandbox profiles | `reference/authoring.md` |
| Copy an agent, playbook or sandbox profile, read its history, or take its parent's update | `reference/authoring.md` (Copies, history and updates) |
| Export or import an agent, playbook or sandbox profile, edit many files at once, or duplicate an agent | `reference/authoring.md` (Export, import and bulk edits) |
| See exactly what a session's agent received | `reference/troubleshooting.md` |
| Make something run on its own — on a pull request, a comment command, a ticket, a Slack alert, a schedule or a webhook | `reference/automations.md` |
| Find out why an automation did not fire, or who a run acted as | `reference/troubleshooting.md`, `reference/automations.md` |
| Work out why something failed | `reference/troubleshooting.md` |
| See or govern which connector tools (Jira, Linear, Sentry, …) a session's agent may call, or why a call was blocked | `reference/connectors.md` |
| Understand what a platform CONCEPT means | the documentation site — below |

The server also carries prompts you can pull directly: `platform_smith_guide` (the full domain
narrative), `diagnose_stuck_launch`, `credential_preflight`, and `authoring_guide`.

## Copies, not merging

An agent definition, playbook or sandbox profile at the organization, a workspace or a project is **its
own complete copy** — what it holds is exactly what its runtime gets; nothing is merged from an outer
level at launch. A copy records its **parent**, and updates never arrive on their own: the parent's owner
**publishes** a version (only published versions count), the copy shows `update_available`, and a person
reviews it with `get_agent_definition_update` and applies it with `apply_agent_definition_update` —
deciding `take` or `keep` for every item both sides changed. One level at a time: a project copy follows
its workspace copy. Every save is a version you can list, compare and restore. Agent files can be
**links** (`link_target`, e.g. `CLAUDE.md` → `AGENTS.md`) or executable (`is_executable`). And
`get_session_snapshot` is the record of what a session's agent actually received, stored before secret
substitution. Details: `reference/authoring.md`.

## Automations

An automation binds a **source** — `github_repo` or `bitbucket_repo` events (pull request, push,
comment, review), comments on `jira`, `linear` or `github` issues, alerts in a `slack` channel, a
cron `schedule` in a timezone, or a signed `webhook` — to a **target**: a coding-agent session, a
workflow, a playbook, or `none` (**record only**: each event creates or links a work item and
nothing runs). Once **enabled** it starts the target **unattended** every time the source fires.

- **Create it switched off** (`"enabled": false`), show the person what it will do and **who it
  acts as**, and only then `enable_automation`. `disable_automation` is the safe stop.
- **Require approval**: `policies.require_approval: {"timeout": "24h"}` holds every run (`held`)
  behind a workspace notification until someone approves it — `approve_automation_run` /
  `decline_automation_run`, or `answer_notification` with `choice` `approve` / `decline` (the same
  first-wins decision). Anyone who can answer the workspace's notifications may approve, including
  whoever raised the event. Unanswered, it expires and never starts.
- **Run detail**: `get_automation_run` reads one run — its approval state and the agent's own
  `verdict` (`pass` / `fail` / `needs_attention`) and summary, which `report_back.pr_comment` posts
  on the pull request.
- **Who a run acts as**: a person who asked — a comment command, a Slack @mention or `/smith` —
  or a pull request's author acts as **themselves**, when their provider account is linked to a
  PlatformSmith user allowed to start the target. Otherwise, and always for schedules, webhooks,
  alerts, pushes and `run_automation_now`, it acts as the **workspace's service principal**.
  `get_automation` shows this read-only as `identity`; `list_automation_runs` shows who each run
  actually ran as and, if it fell back, why (`identity_note`).
- **Commands**: people type `<prefix> <verb> [args]` in a comment (e.g. `/ps review security`);
  the verb is the automation's `command_verb`, the args reach the target. Read the prefix with
  `get_workspace_command_prefix` before telling anyone what to type.
- **Work items**: `work_item: {"mode": "link"}` attaches each run to one work item per pull
  request, issue, alert or schedule fire.
- **"Review every PR"** is configuration, not code: a `github_repo` automation on
  `pull_request.opened` and `pull_request.synchronize` targeting a review playbook or an agent
  session. A new push supersedes the review in flight; the platform's own pushes never re-trigger it.
- **Schedules**: check the expression with `preview_cron` (cron + IANA timezone) first;
  `run_automation_now` fires one immediately to try it.
- **Credentials warn, they never refuse**: a missing shared coding-agent credential is reported
  when you save, and runs acting as the service principal then fail as `no_credential`. Save-time
  `warnings` (`duplicate_verb`, `prefix_mismatch`) do not block the save — pass them on.
- **@mentions and `/smith`** in Slack run on a **built-in** automation each workspace has: switch
  it on or off, never edit or delete it.
- **"Why didn't it fire?"** — `list_automation_deliveries`: one envelope-only row per event, with
  the outcome and the reason it was dropped. What it started, and who it ran as, is
  `list_automation_runs`. No deliveries at all from a repository usually means its
  GitHub/Bitbucket **account** does not have **Receives events** on — `get_integration_account`
  (new git accounts start with it off).
- A webhook with a **generated** secret, and rotating a secret, are done on the Automations page in
  the app — the secret is shown once there, and no tool returns a secret.

Recipes and the full reason table: `reference/automations.md`.

## Connectors and guardrails

A **connector** is a vendor's MCP server (Jira, Linear, GitHub, Sentry, …) whose tools a
coding-agent session can call through the platform; `platform` is the built-in one. Connectors are
not integrations — those are the credentials the platform itself uses. A session gets a connector
tool only when the agent definition's `.platform-smith/tools.json` lists it, the organisation has
enabled the connector and reviewed the tool, no **guardrail** blocks it (guardrails only restrict;
the strictest wins), and **wiring** picks an account (environment override → workspace default →
the only account assigned).

- `preview_session_tools` shows what a session would get, and why, before anything launches.
- "Why can't my agent call X?" — `get_session_tools`, then `explain_tool_decision` on its audit row.
  The audit (`list_tool_audit`) is metadata only: never a call's arguments or result.
- Enabling connectors, reviewing tools, guardrails and account assignment are **admin actions** —
  ask the person first. Connecting an account is done in the UI under **Org → Connectors**.

Details: `reference/connectors.md`.

## The documentation site — tools for state, docs for concepts

This skill and the tool inventory teach you the platform's **verbs**. They do not teach its
**vocabulary**: a tool schema tells you that `launch` takes an `environment_uuid`, and nothing tells
you what an environment *is*, how it relates to a project, or when you want a new one rather than a
new sandbox. Inferring that from a schema is exactly where an agent guesses.

The documentation is published in an agent-readable form. **Start here:**

```
https://docs.platformsmith.com/llms.txt
```

It is a small table of contents — one line per section (concepts, environments, agent definitions,
credentials, artifacts, Dockerfile, API and automation, getting started, …) with absolute URLs — so it is
cheap to fetch speculatively and it links onward to only what you need. Every page also has a
markdown twin: append `.md` to a page URL and you get the source rather than HTML you have to parse.
There is a `llms-full.txt` with the entire corpus, but prefer `llms.txt` and follow its links —
the full file is large and mostly irrelevant to any one question.

**The division of labour, which decides where to look:**

| Question | Answer from |
|---|---|
| What *is* an environment / agent definition / sandbox profile? | **the docs** |
| Which environments does this workspace have, right now? | **a tool** (`list_environments`) |
| What does `on_failure` mean, and what are my options? | **the docs** |
| What is this profile's `on_failure` set to? | **a tool** (`get_sandbox_profile`) |
| Why would I use a playbook instead of a workflow? | **the docs** |
| Did this playbook run succeed? | **a tool** (`get_playbook_run`) |

Tools are the only truth about **state** — the docs describe how the product works in general and can
lag a deployment. So never answer a question about *this tenant* from the docs, and never guess a
*concept* from a tool schema when one fetch would tell you.

## Rules that are not negotiable

- **Pointing this at real tenant content is a data decision, not a config step.** The read surface
  includes session transcripts, prompts, agent output, source code, user names and email addresses,
  and everything you read enters this model's context. While you are learning the tools, prefer a
  workspace whose data you are free to expose. See `reference/connecting.md`.

- **Secrets are metadata-only.** You can list schemas, names, refs and status, attach refs, and
  **write** a value with `create_secret` / `update_secret`. You can **never read a decrypted value
  back** — there is no tool, and asking for one differently will not produce one.
- **Administration is not exposed, deliberately.** No PAT or workspace-token minting, no RBAC, no
  user creation, no integration setup (connecting GitHub, Bitbucket, Slack, Jira or Linear,
  Receives events, workspace assignment, linking people), no connecting or removing a connector
  account, no changing the command prefix, no workspace/project/environment CRUD. (Connector
  *governance* over accounts that already exist — guardrails, tool reviews, wiring — is exposed, as
  admin actions; see above.) If you need one of
  these, **say exactly what is blocked and ask the human to do it in the UI** — that is more useful
  than a workaround, and there is no workaround.
- **There is no push.** Session events are a cursor-poll loop; there is no streaming and no
  blocking wait. See `reference/running-work.md`.
- **The resource list looks empty, and that is correct.** Artifacts ARE readable
  as MCP resources, but they are published as URI *templates* — the readable set is per-tenant, so
  there is nothing tenant-independent to enumerate. If your client reports "no resources", it is
  reading the wrong list. Read one directly at
  `psmcp://sessions/{session_uuid}/artifacts/{artifact_version_uuid}` (ids come from
  `list_session_artifacts`). The `get_artifact` tool does the same thing inline if you prefer.

## When this skill and the server disagree, the server wins

`reference/tools.md` is generated from one build of ps-mcp and can lag the server you are talking to.
The live `tools/list` is authoritative for what exists, and `get_workflow_task_catalog` is
authoritative for workflow node types and their input schemas — trust it over any node list written
down here or anywhere else.
