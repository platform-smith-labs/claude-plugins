---
name: platform-smith
description: Drive the PlatformSmith control plane over MCP (ps-mcp) — orient in a tenant, launch runtimes and coding-agent sessions, follow them to completion, read results, author agent definitions, workflows and playbooks, and diagnose failures. Use whenever the ps-mcp MCP server is connected and the task involves PlatformSmith workspaces, projects, environments, runtimes, sessions, workflows, playbooks, work items or artifacts.
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
| Author agent definitions, workflows, playbooks, sandbox profiles | `reference/authoring.md` |
| Work out why something failed | `reference/troubleshooting.md` |

The server also carries prompts you can pull directly: `platform_smith_guide` (the full domain
narrative), `diagnose_stuck_launch`, `credential_preflight`, and `authoring_guide`.

## Rules that are not negotiable

- **Pointing this at real tenant content is a data decision, not a config step.** The read surface
  includes session transcripts, prompts, agent output, source code, user names and email addresses,
  and everything you read enters this model's context. While you are learning the tools, prefer a
  workspace whose data you are free to expose. See `reference/connecting.md`.

- **Secrets are metadata-only.** You can list schemas, names, refs and status, attach refs, and
  **write** a value with `create_secret` / `update_secret`. You can **never read a decrypted value
  back** — there is no tool, and asking for one differently will not produce one.
- **Administration is not exposed, deliberately.** No PAT or workspace-token minting, no RBAC, no
  user creation, no connector/Slack setup, no workspace/project/environment CRUD. If you need one of
  these, **say exactly what is blocked and ask the human to do it in the UI** — that is more useful
  than a workaround, and there is no workaround.
- **There is no push.** Session events are a cursor-poll loop; there is no streaming and no
  blocking wait. See `reference/running-work.md`.
- **The resource list looks empty, and that is correct.** Artifacts and resolved files ARE readable
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
