# SPAWN.md - GitMultiAgent Subagent Delegation

This file maps OpenClaw-style subagent delegation onto the GitMultiAgent mechanics
that already exist in this repository.

## Actual GitMultiAgent Mechanics

GitMultiAgent does not schedule work from task type alone.

The live objects are:

- the root agent: an explicit interactive agent from `.git-multiagent/team.toml`
- the workers: named queued workers from `.git-multiagent/team.toml`
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

OpenClaw `sessions_spawn` maps to GitMultiAgent task/job dispatch, not agent
creation.

| OpenClaw behavior | GitMultiAgent behavior in this repo |
| --- | --- |
| Parent asks for delegated work | Root agent creates or updates a task |
| Parent delegation decision | Planner job created by `task-create` |
| Child session | A subagent job on the task |
| Child initial user message | Task spec and job spec |
| Child system prompt | `.git-multiagent/roles/subagent.md` |
| Child result | `task-comment`, job log, transcript; planner may later write `task-result` |
| Child runtime mode | `mode` and `options` fields in `.git-multiagent/team.toml` |
| Spawned agent object | No equivalent during dispatch; agents are standing team workers |

## Dispatch In This Repo

For this repo's OpenClaw-like setup, the explicit interactive `agent` is the
root agent. When it wants subagent help, it uses the existing GitMultiAgent task
mechanics:

1. Write a task spec with `type: subagents`.
2. Create the task with the normal GitMultiAgent task-create path.
3. Let the initial `role=planner` job plan the task and create one or more
   `role=subagent` jobs attached to that task.
4. Continue or wait according to whether the user-facing work is blocking.

The task spec carries the shared context. Each job spec carries one bounded
assignment for one subagent worker.

Current layer-1 `task-create` creates `<task-id>-plan` with `role=planner`.
That planner job is the normal GitMultiAgent entry point for turning a task into
subagent jobs. It is not a spawned subagent.

## Task Spec Shape

Use a normal GitMultiAgent task spec. Put the subagent type in the spec because
GitMultiAgent currently has no first-class task type field:

```markdown
---
type: subagents
parent: <parent-task-or-agent-session>
blocking: true|false
---

# <task name>

## Objective
<what the delegated work should accomplish>

## Context
<only the context needed by the subagent>

## Expected Output
<the result shape needed by the root agent>

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
jobs or close the task with `task-result`. The root agent reads task updates,
inspects referenced evidence when needed, and synthesizes the user-facing
response. Subagent output is evidence.

## Planner To Root Agent Handoff

GitMultiAgent has planner notification jobs for worker-to-planner coordination.
It does not have a separate root-agent notifier in this architecture.

When the planner reaches a user-visible decision, it should first update the
task record with `task-comment` and then record completion, failure, or blocked
state with the normal task result path. The interactive root agent is explicit
in `.git-multiagent/team.toml`; it can inspect task updates through heartbeat,
dashboard chat, or direct prompts. Do not create `role=agent` notification jobs.

The task record is the handoff. It does not replace task state because it is
task state.

## Blocking And Non-Blocking

For blocking delegation, the root agent waits for the relevant planner/task update
before answering. The planner may wait on subagent evidence before closing or
requesting more work.

For non-blocking delegation, the root agent records the delegated task/job ids and
continues. Task updates bring completed or blocked work back to the root agent.

## What Not To Do

- Do not create agents as part of delegation.
- Do not call or invent `spawn_agent`.
- Do not edit `.git-multiagent/team.toml` during delegation.
- Do not rely on `type: subagents` alone to run work; the planner must create
  `role=subagent` jobs.
- Do not delete task history, job logs, transcripts, or agent records after a
  subagent finishes.
