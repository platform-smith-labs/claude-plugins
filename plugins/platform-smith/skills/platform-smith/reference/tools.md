# ps-mcp tool reference

<!-- GENERATED from internal/tools/*specs.go by GenerateSkillInventory — DO NOT EDIT BY HAND.
     Regenerate: go test ./internal/tools/ -run TestWriteSkillInventory -v (with PS_WRITE_SKILL=1)
     TestGeneratedSkillInventoryIsCurrent fails the build if this file drifts. -->

143 tools. **Effect** says whether a call is a read, a write, or destructive — it mirrors the MCP annotation the server sends, so a harness that auto-approves reads is using the same signal.

⚠️ This inventory is a SNAPSHOT of one build, and it lists every tool ps-mcp CAN serve — not necessarily what the server you are connected to DOES serve. A deployment can hide whole feature areas (work items, workflows, artifacts) behind its feature curtain, and those tools are then absent from `tools/list` and refused if called. That is configuration, not an outage, and not a version mismatch: do not retry and do not report it as a bug. The live `tools/list` always outranks this file, and `get_workflow_task_catalog` outranks any workflow node list written down anywhere. If they disagree, believe the server.

## agent-definitions

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `create_agent_definition` | write | `project_uuid` | Create an agent definition in a project. |
| `create_agent_definition_file` | write | `agent_definition_uuid`, `project_uuid` | Create a file body on an agent definition. |
| `create_scoped_agent_definition` | write | `scope` | Create an agent definition at the given scope (scope=project\|workspace\|company, scope_id=that container's UUID; omit scope_id for scope=company — the company is taken from your identity). |
| `create_scoped_agent_definition_file` | write | `agent_definition_uuid` | Create a file body on an agent definition AT ANY SCOPE (company, workspace, or project — no scope/scope_id needed, unlike create_scoped_agent_definition: the target definition's scope is already fixed). |
| `delete_agent_definition` | **destructive** | `agent_definition_uuid` | Delete an agent definition. |
| `delete_agent_definition_file` | **destructive** | `agent_definition_uuid`, `file_uuid`, `project_uuid` | Delete an agent-definition file body by file_uuid. |
| `get_agent_definition` | read | `agent_definition_uuid` | Get an agent definition by UUID. |
| `get_agent_definition_file` | read | `agent_definition_uuid`, `file_uuid`, `project_uuid` | Get one agent-definition file body by file_uuid. |
| `get_agent_definition_resolved_files` | read | `agent_definition_uuid` | Get an agent definition's RESOLVED files — the effective set after inheritance and merge, i.e. |
| `get_detected_secrets` | read | `project_uuid` | List secrets detected in a project's agent definitions (references/metadata only — never decrypted values). |
| `list_agent_definition_files` | read | `agent_definition_uuid` | List an agent definition's files. |
| `list_agent_definitions` | read | `project_uuid` | List agent definitions in a project. |
| `list_scoped_agent_definitions` | read | `scope` | List agent definitions at the given scope (scope=project\|workspace\|company). |
| `preview_agent_definition` | read | `project_uuid` | Preview the effective (merged) agent-definition config for a project. |
| `reset_agent_definition` | **destructive** | `agent_definition_uuid` | Reset an agent definition to default. |
| `update_agent_definition` | write | `agent_definition_uuid` | Replace an agent definition (full PUT update, not a partial patch). |
| `update_agent_definition_file` | write | `agent_definition_uuid`, `file_uuid`, `project_uuid` | Replace an agent-definition file body (full PUT). |

## agent-profiles

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_agent_profile` | read | `type` | Get an agent profile by type. |
| `list_agent_profiles` | read | — | List available agent profiles (distinct from agent definitions). |

## artifacts

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_artifact` | read | `artifact_version_uuid`, `session_uuid` | Fetch ONE artifact by its artifact_version_uuid (from list_session_artifacts). |
| `get_scoped_artifact` | read | `artifact_version_uuid`, `scope`, `scope_id` | Fetch ONE project- or workspace-scoped artifact by its artifact_version_uuid (from list_scoped_artifacts). |
| `list_scoped_artifacts` | read | `scope`, `scope_id` | List artifacts at project or workspace scope (scope=project\|workspace, scope_id=that container's UUID). |
| `list_session_artifacts` | read | `session_uuid` | List a session's artifacts. |

## audit

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_workspace_audit` | read | `workspace_uuid` | The workspace's audit trail — what happened here, and who did it. |

## available

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `list_available_sandbox_profiles` | read | `scope`, `scope_id` | List the sandbox profiles a launch can choose (scope=project\|workspace, scope_id=that container's UUID). |

## connections

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_integration_connection` | read | `connection_uuid` | Get one integration connection's metadata and status (read-only, never a decrypted value) — is it enabled, and what auth_type does it carry? |
| `list_git_connection_repos` | read | `connection_uuid` | List the repositories a git connection can see (read-only). |
| `list_git_connections` | read | — | List the company's git connections (read-only). |
| `list_integration_connections` | read | — | List the company's integration connections — including the claude_code / codex credentials a runtime needs (read-only, never a decrypted value). |
| `list_workspace_git_connections` | read | `workspace_uuid` | List the git connections attached to a workspace (read-only). |

## controllers

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_controller` | read | `name` | Get one controller by name — check it is connected before blaming a launch failure on the runtime. |
| `get_workspace_controllers_summary` | read | `workspace_uuid` | Summary of controller health for a workspace. |
| `list_controllers` | read | — | List the controllers (container hosts) registered for the caller's company. |

## conversations

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `create_conversation` | write | `workspace_uuid` | Start an agent-to-agent conversation in a workspace. |
| `get_conversation` | read | `conversation_uuid`, `workspace_uuid` | Get one A2A conversation by UUID. |
| `list_conversation_delivery_receipts` | read | `conversation_uuid`, `workspace_uuid` | Delivery receipts for an A2A conversation — did the peer actually RECEIVE the message, as distinct from whether it was posted? The difference between "my message is in the log" and "my message arrived", which is the difference between… |
| `list_conversation_messages` | read | `conversation_uuid`, `workspace_uuid` | Read an A2A conversation's append-only message log — what the agents actually said. |
| `list_conversation_participants` | read | `conversation_uuid`, `workspace_uuid` | List who is taking part in an A2A conversation. |
| `list_conversations` | read | `workspace_uuid` | List the A2A conversations in a workspace. |
| `list_recent_conversations` | read | `workspace_uuid` | The workspace's most recent A2A conversations — the orientation view when you know something was discussed but not which conversation. |
| `send_conversation_message` | write | `conversation_uuid`, `workspace_uuid` | Post a message into an A2A conversation. |

## default

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `set_sandbox_profile_default` | write | `sandbox_profile_uuid` | Mark or unmark a sandbox profile as its scope's default. |

## environments

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_environment` | read | `environment_uuid` | Get a single environment by UUID (read-only). |
| `list_environments` | read | `workspace_uuid` | List environments in a workspace (read-only). |

## git-link

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_project_git_link` | read | `project_uuid` | Get a project's git link — which repo and branch it is bound to (read-only). |

## integrations

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `list_workspace_integrations` | read | `workspace_uuid` | List the integration connections attached to a workspace (read-only). |

## launch

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `launch` | **destructive** | — | Launch a runtime for a project/environment (modern launch path). |

## launches

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_launch` | read | `instance_uuid` | Get a launch (runtime instance) by instance UUID. |
| `list_launch_attempts` | read | `instance_uuid` | List the attempts for a launch (runtime instance). |
| `list_launches` | read | — | List launches (runtime instances) for the caller's company. |

## members

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `list_workspace_members` | read | `workspace_uuid` | List the members of a workspace — the actors you can assign work to. |

## permissions

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_my_permissions` | read | — | List the permissions the calling identity actually holds. |

## platformsmith-attempts

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_attempt_files` | read | `attempt_uuid`, `project_uuid` | Get the files produced by ONE specific attempt (attempt_uuid from list_platformsmith_attempts). |
| `get_latest_attempt_files` | read | `project_uuid` | Get the files produced by the MOST RECENT attempt on a project — the shortcut when you just want 'what did the last run change?'. |
| `list_platformsmith_attempts` | read | `project_uuid` | List the PlatformSmith attempts recorded against a project — the history of automated coding attempts, with their status and PR links. |

## playbook-runs

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_playbook_run` | read | `playbook_run_uuid` | Get one playbook run — its status and verdict. |
| `list_workspace_playbook_runs` | read | `workspace_uuid` | List every playbook run in a workspace, newest-first. |
| `start_playbook_run` | **destructive** | — | Start a playbook run. |

## playbooks

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `create_playbook` | write | — | Create a playbook definition. |
| `get_playbook` | read | `playbook_uuid` | Get one playbook by UUID. |
| `list_playbook_runs` | read | `playbook_uuid` | List the runs of one playbook — the history for that playbook. |
| `list_workspace_playbooks` | read | `workspace_uuid` | List the playbooks available in a workspace. |
| `update_playbook` | write | `playbook_uuid` | Update a playbook definition. |

## pr-url

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `set_platformsmith_attempt_pr_url` | write | `attempt_uuid`, `project_uuid` | Record the PR URL an attempt produced. |

## projects

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_project` | read | `project_uuid` | Get a single project by UUID. |
| `list_projects` | read | `workspace_uuid` | List projects in a workspace. |

## readiness

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_readiness` | read | — | Check whether a target is READY to launch before launching into a failure. |

## restore

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `restore_sandbox_profile` | **destructive** | `sandbox_profile_uuid` | Restore an archived sandbox profile. |

## runtimes

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `delete_runtime` | **destructive** | `runtime_uuid` | Delete a runtime. |
| `get_runtime` | read | `runtime_uuid` | Get a single runtime by its runtime_uuid. |
| `list_runtimes` | read | — | List the caller company's runtimes. |
| `list_workspace_runtimes` | read | `workspace_uuid` | List runtimes scoped to one workspace — the narrower view when the unscoped list_runtimes is too noisy. |
| `restart_runtime` | **destructive** | `runtime_uuid` | Restart a runtime. |
| `stop_runtime` | **destructive** | `runtime_instance_uuid` | Stop a sandbox and every session running on it. |

## sandbox-profiles

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `archive_sandbox_profile` | **destructive** | `sandbox_profile_uuid` | Archive a sandbox profile (nothing is deleted; restore_sandbox_profile brings it back). |
| `clone_sandbox_profile` | write | `sandbox_profile_uuid` | Copy a sandbox profile, with its secret refs, into the same scope or a narrower one. |
| `create_sandbox_profile` | write | `scope` | Create a sandbox profile at a scope (scope=project\|workspace\|company; omit scope_id for company). |
| `get_sandbox_profile` | read | `sandbox_profile_uuid` | Get one sandbox profile, including its script and its secret_refs (names and env var names only — never values). |
| `list_sandbox_profiles` | read | `scope` | List the sandbox profiles OWNED by one scope (scope=project\|workspace\|company; omit scope_id for company). |
| `update_sandbox_profile` | write | `sandbox_profile_uuid` | Update a sandbox profile. |

## secret-refs

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `add_agent_definition_secret_ref` | write | `agent_definition_uuid` | Attach a secret ref to an agent definition (ref only, never a value). |
| `add_sandbox_profile_secret_ref` | write | `sandbox_profile_uuid` | Attach a secret, by name, to a sandbox profile (a reference, never a value). |
| `add_workflow_definition_secret_ref` | write | `workflow_definition_uuid` | Attach a secret ref to a workflow definition (ref only, never a value) — the exact counterpart of add_agent_definition_secret_ref, which has shipped since v1. |
| `get_agent_definition_secret_refs_status` | read | `agent_definition_uuid` | Check whether an agent definition's secret refs actually RESOLVE (present / missing / forbidden). |
| `get_sandbox_profile_secret_refs_status` | read | `sandbox_profile_uuid` | Check whether a sandbox profile's secret refs RESOLVE for a launch context (pass the project_uuid you will launch, and workspace_uuid / environment_uuid when known). |
| `get_workflow_definition_secret_refs_status` | read | `workflow_definition_uuid` | Check whether a workflow definition's secret refs actually RESOLVE (present / missing / forbidden). |
| `list_agent_definition_secret_refs` | read | `agent_definition_uuid` | List an agent definition's secret refs (references only — never values). |
| `list_sandbox_profile_secret_refs` | read | `sandbox_profile_uuid` | List a sandbox profile's secret refs: {sandbox_profile_secret_ref_uuid, secret_name, inject_as, env_name}. |
| `list_workflow_definition_secret_refs` | read | `workflow_definition_uuid` | List a workflow definition's secret refs (references only — never values). |
| `remove_sandbox_profile_secret_ref` | **destructive** | `sandbox_profile_uuid`, `secret_ref_uuid` | Detach a secret ref from a sandbox profile. |

## secret-schemas

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_secret_schema` | read | `secret_type` | Get a secret schema by type. |
| `list_secret_schemas` | read | — | List secret schemas (the shapes secrets/definitions expect). |

## secrets

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `create_secret` | write | `scope` | Create a secret at the given scope (scope=project\|workspace\|company, scope_id=that container's UUID; omit scope_id for scope=company — the company is taken from your identity). |
| `get_secret_descendants` | read | `secret_store_uuid` | List the secrets that INHERIT from this one down the scope chain (company → workspace → project). |
| `get_secret_metadata` | read | `secret_store_uuid` | Get a secret's metadata (name/ref/status/schema). |
| `get_secret_usage` | read | `secret_store_uuid` | Show where a secret is used. |
| `list_project_secrets` | read | `project_uuid` | List a project's secrets (metadata only — no decrypted values). |
| `update_secret` | write | `secret_store_uuid` | Update a secret — ROTATION. |

## sessions

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `create_session` | **destructive** | — | Create a runtime session directly (the non-workflow path to a working session). |
| `get_session` | read | `name` | Get a session by name. |
| `get_session_events` | read | `name` | Cursor-poll a session's events. |
| `list_conversation_sessions` | read | `conversation_uuid`, `workspace_uuid` | List the runtime sessions backing an A2A conversation's participants — the bridge from 'what was said' to 'which session said it', then on to that session's events and artifacts. |
| `list_sessions` | read | — | List runtime sessions the caller can access. |
| `list_workspace_sessions` | read | `workspace_uuid` | List sessions scoped to one workspace. |
| `send_session_input` | write | `name` | Send input to a running session (e.g. |
| `stop_session` | **destructive** | `name` | Stop a session's coding agent. |

## signal-decisions

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `create_workflow_approval` | **destructive** | — | DECIDE a human approval gate — an await-signal(shape=approval) node that has parked and surfaced in the workflow inbox. |

## task-completions

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `create_task_completion` | **destructive** | — | Record a task completion (mark a workflow task done). |

## tasks

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_task` | read | `task_uuid` | Get one task by UUID — this is how you follow up the task_uuid returned by launch or spawn_runtime. |
| `get_task_responses` | read | `task_uuid` | Get the response payload of a completed task. |
| `list_tasks` | read | — | List the caller company's tasks (the async job records behind launch/spawn_runtime). |
| `spawn_runtime` | **destructive** | — | Low-level spawn via the unified tasks endpoint. |

## terminate

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `terminate_workflow_execution` | **destructive** | `execution_id` | Stop a running workflow execution (issue 0042 finding 0006 — a stuck fork/join previously had no way to be ended short of the platform UI). |

## work-items

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `add_work_item_comment` | write | `work_item_uuid`, `workspace_uuid` | Add a comment to a work item — the place to record why a decision was made, or what an attempt found, so the next reader is not guessing. |
| `assign_work_item` | write | `work_item_uuid`, `workspace_uuid` | Set or clear who owns a work item. |
| `create_child_work_item` | write | `work_item_uuid`, `workspace_uuid` | Decompose a work item — create a child under it. |
| `create_work_item` | write | `workspace_uuid` | Create a root work item. |
| `get_work_item` | read | `work_item_uuid`, `workspace_uuid` | One work item in depth: its ancestry, the subtree beneath it, its dependency DAG, every attempt against it, its artifacts and its activity. |
| `get_work_item_ready` | read | `work_item_uuid`, `workspace_uuid` | Is this work item READY to start — i.e. |
| `get_workspace_work_item_rollup` | read | `workspace_uuid` | The workspace's work-item roll-up: what is queued, running, blocked, verified or failed across every project, with the attempts behind each. |
| `remove_work_item_dependency` | **destructive** | `work_item_uuid`, `workspace_uuid` | Remove an ordering dependency. |
| `set_work_item_rank` | write | `work_item_uuid`, `workspace_uuid` | Set a work item's ordering weight among its siblings. |
| `set_work_item_state` | write | `work_item_uuid`, `workspace_uuid` | Move a work item to a new lifecycle state. |
| `update_work_item` | write | `work_item_uuid`, `workspace_uuid` | Edit a work item's title, description or goal. |

## workflow-definitions

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `clone_workflow_definition` | write | `workflow_definition_uuid` | Clone an existing workflow definition. |
| `create_scoped_workflow_definition` | write | `scope` | Create a workflow definition at the given scope (scope=project\|workspace\|company, scope_id=that container's UUID; omit scope_id for scope=company — the company is taken from your identity). |
| `create_workflow_definition` | write | `project_uuid` | Create a workflow definition in a project. |
| `delete_workflow_definition` | **destructive** | `workflow_definition_uuid` | Delete a workflow definition. |
| `get_workflow_definition` | read | `workflow_definition_uuid` | Get a workflow definition by UUID. |
| `list_scoped_workflow_definitions` | read | `scope` | List workflow definitions at the given scope (scope=project\|workspace\|company). |
| `list_workflow_definitions` | read | `project_uuid` | List workflow definitions in a project. |
| `publish_workflow_definition` | write | `workflow_definition_uuid` | Publish a workflow definition (draft → live). |
| `update_workflow_definition` | write | `workflow_definition_uuid` | Replace a workflow definition (full PUT update). |
| `validate_workflow_definition` | write | — | Validate a DRAFT conductor_json BEFORE create/publish (no persistence). |

## workflow-executions

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_workflow_execution` | read | `execution_id` | Get a workflow execution (run) by id. |
| `list_workflow_executions` | read | — | List workflow executions (runs). |
| `run_workflow` | **destructive** | — | Start a workflow run (create a workflow execution). |
| `run_workflow_in_workspace` | **destructive** | `workspace_uuid` | Start a workflow run attributed to a specific workspace. |

## workflow-inbox

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_workflow_inbox` | read | `scope`, `scope_id` | List the workflow inbox (pending items awaiting action) at the given scope (scope=workspace\|project, scope_id=that container's UUID). |

## workflow-task-catalog

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_workflow_task_catalog` | read | — | List the workflow task catalog (the task types available to compose into workflows). |

## workspaces

| Tool | Effect | Required args | Purpose |
|---|---|---|---|
| `get_workspace` | read | `workspace_uuid` | Get a single workspace by UUID. |
| `get_workspace_agents_summary` | read | `workspace_uuid` | Summary of agent activity across a workspace — the fastest orientation on what is running before you drill into individual sessions. |
| `list_workspaces` | read | — | List the workspaces the caller's company can access. |
