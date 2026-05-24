# SPAWN.md - GitAgents Subagent Delegation

This file maps OpenClaw-style subagent delegation onto the GitAgents mechanics
that already exist in this repository.

## Actual GitAgents Mechanics

GitAgents does not schedule work from task type alone.

The live objects are:

- the console: an interactive agent, not listed in `team.toml`
- the team: named queued workers from `.git-agents/team.toml`
- a task: shared long-lived context under `tasks/<task-id>/`
- a job: the runnable unit under `jobs/<job-id>/`
- a role: the queue key a worker waits on and claims

Queued workers do not claim tasks. They run `job-wait -r <role>`, claim one
pending job for that role, process that job, write task comments/logs, close or
release the job, and exit. The supervisor restarts them.

Therefore, a subagent delegation eventually needs a `role=subagent` job. A
`type: subagents` task without a subagent job is only metadata; it will not wake
a subagent worker.

## Mapping

OpenClaw `sessions_spawn` maps to GitAgents task/job dispatch, not agent
creation.

| OpenClaw behavior | GitAgents behavior in this repo |
| --- | --- |
| Parent asks for delegated work | Console creates or updates a task |
| Parent delegation decision | Planner job created by `task-create` |
| Child session | A subagent job on the task |
| Child initial user message | Task spec and job spec |
| Child system prompt | `.git-agents/roles/subagent.md` |
| Child result | `task-comment`, job log, transcript; planner may later write `task-result` |
| Child model/profile | `engine` and `model` fields in `.git-agents/team.toml` |
| Spawned agent object | No equivalent during dispatch; agents are standing team workers |

## Dispatch In This Repo

For this repo's OpenClaw-like setup, the console is the root agent. When it wants
subagent help, it uses the existing GitAgents console authority:

1. Write a task spec with `type: subagents`.
2. Create the task with the normal GitAgents task-create path.
3. Let the initial `role=planner` job plan the task and create one or more
   `role=subagent` jobs attached to that task.
4. Continue or wait according to whether the user-facing work is blocking.

The task spec carries the shared context. Each job spec carries one bounded
assignment for one subagent worker.

Current layer-1 `task-create` creates `<task-id>-plan` with `role=planner`.
That planner job is the normal GitAgents entry point for turning a task into
subagent jobs. It is not a spawned subagent.

## Task Spec Shape

Use a normal GitAgents task spec. Put the subagent type in the spec because
GitAgents currently has no first-class task type field:

```markdown
---
type: subagents
parent: <parent-task-or-console-session>
blocking: true|false
---

# <task name>

## Objective
<what the delegated work should accomplish>

## Context
<only the context needed by the subagent>

## Expected Output
<the result shape needed by the console/root>

## Scope
<allowed files, commands, repositories, and non-goals>

## Verification
<evidence or checks expected before reporting back>
```

## Job Spec Shape

Each subagent job should be smaller than the task and executable by one worker:

```markdown
# Subagent Job: <short name>

## Task
<task-id>

## Assignment
<specific bounded work>

## Context
<task-local context needed for this worker>

## Output
Report with a task comment beginning `Subagent result:`.

## Verification
<what evidence to collect or checks to run>
```

## Communication

The task is the channel.

Subagents report through:

- `task-comment` for progress and final findings
- job logs for detailed evidence
- transcripts for the full run trail
- a `role=planner` notification job before terminal job transition
- job status for completion, release, or failure

The planner reads subagent reports and decides whether to create more subagent
jobs or close the task with `task-result`. The console reads task updates,
inspects referenced evidence when needed, and synthesizes the user-facing
response. Subagent output is evidence.

## Planner To Console Notification

GitAgents has planner notification jobs for worker-to-planner coordination and
console notification jobs for planner-to-console coordination.

The console is not a queued team worker, but `role=console` jobs are consumed by
the GitAgents console notifier and forwarded to the interactive console. Do not
add a console agent to `.git-agents/team.toml`.

When planner reaches a user-visible decision, it should first update the task
record, then create a console notification job on the same task.

This is the GitAgents equivalent of a parent-session completion event. It does
not replace task state. It only wakes the console so the console can inspect the
task and respond to the user.

## Blocking And Non-Blocking

For blocking delegation, the console waits for the relevant planner/task update
before answering. The planner may wait on subagent evidence before closing or
requesting more work.

For non-blocking delegation, the console records the delegated task/job ids and
continues. The planner-to-console notification brings completed or blocked work
back to the console.

## What Not To Do

- Do not create agents as part of delegation.
- Do not call or invent `spawn_agent`.
- Do not edit `.git-agents/team.toml` during delegation.
- Do not rely on `type: subagents` alone to run work; the planner must create
  `role=subagent` jobs.
- Do not delete task history, job logs, transcripts, or agent records after a
  subagent finishes.
