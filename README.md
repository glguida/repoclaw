# RepoClaw

RepoClaw is an OpenClaw-like resident agent living in a git repository.

It gives a repo its own persistent AI presence: an interactive root agent you can talk to,
memory files, heartbeat behavior, task history, and a small worker team for
planning and delegated work.

RepoClaw is implemented with GitMultiAgent. GitMultiAgent provides the process model,
tasks, jobs, roles, state directory, and interactive agent plumbing. RepoClaw is the
resident agent layer that lives inside a repository and learns that repository
over time.

## One Repo, One Claw

You can have a separate RepoClaw resident agent for each git repository.

That is the intended use. Each project can have its own agent identity, memory,
notes, task history, and local habits. Different repositories should not share
the same working memory unless you explicitly want that.

A RepoClaw instance belongs to the repository it serves. It reads that repo's
root `AGENTS.md`, keeps runtime state under that repo's `.git-multiagent/state/`,
and can grow project-specific files such as `MEMORY.md`, `USER.md`,
`IDENTITY.md`, and daily notes.

This repository is the reusable base. Concrete instances should live in their
own repositories.

## What You Get

RepoClaw gives a git repo:

- an interactive root agent you can talk to from the terminal;
- first-run bootstrap instructions for shaping the agent identity;
- persistent Markdown memory and notes;
- configured heartbeat pings so the interactive root agent can notice work without constant prompting;
- a planner role for organizing delegated work;
- standing subagents for bounded background investigation or checks;
- task/job based communication through GitMultiAgent;
- restart recovery so the interactive root agent can read its transcript and resume context.

It is useful when you want a repository to have an ongoing agent presence rather
than only ad hoc chat sessions.

## Basic Use

Start the workspace:

```sh
git multiagent start
```

Talk to the interactive root agent:

```sh
git multiagent prompt "hello"
```

Check status:

```sh
git multiagent status
```

Stop it:

```sh
git multiagent stop
```

Restart an already-running workspace:

```sh
git multiagent start --restart
```

## First Run

On a fresh instance, the interactive root agent should follow `BOOTSTRAP.md`.

That bootstraps the workspace identity: who the agent is, who the human is, what
tone to use, and what should be written into memory. After bootstrap, the
instance can commit its own identity and memory files in its own repo.

RepoClaw itself should stay generic. Do not commit one project's identity or
memory back into this base unless you want that behavior to become the default
for every future instance.

## Using It With Another Git Repo

Create a RepoClaw instance for a project by copying or cloning this base into
the project workspace you want the agent to serve. Then start GitMultiAgent from
that repo root.

Typical flow:

1. Create or clone a repo-specific RepoClaw workspace.
2. Start it with `git multiagent start`.
3. Talk to it with `git multiagent prompt`.
4. Let bootstrap create the repo-specific identity and memory.
5. Commit those repo-specific files in that instance repo.

The important rule is separation: each target repository can have its own
RepoClaw, with its own `.git-multiagent/state/`, memory, and habits.

## Files You Edit

The main user-facing files are:

- `AGENTS.md`: the root instruction entry point;
- `BOOTSTRAP.md`: first-run identity setup;
- `SOUL.md`: default identity and continuity guidance;
- `HEARTBEAT.md`: heartbeat notes;
- `TOOLS.md`: local tool notes;
- `SPAWN.md`: how delegation maps to GitMultiAgent tasks and jobs.

Roles live under `.git-multiagent/roles/`.

Most day-to-day customization happens in the root Markdown files and in
instance-specific memory files.

## Runtime State

Runtime state is generated under:

```text
.git-multiagent/state/
```

That directory contains pids, transcripts, job state, logs, and Pi session data.
It is intentionally ignored by git.

Useful debug files include:

```text
.git-multiagent/state/agents/agent/transcript.log
.git-multiagent/state/agents/agent/error.log
.git-multiagent/state/logs/supervisor.log
.git-multiagent/state/logs/heartbeat.log
```

Do not commit `.git-multiagent/state/`.

## Heartbeats

Heartbeat is configured per interactive agent in `.git-multiagent/team.toml`.
This base RepoClaw instance sets the root agent heartbeat to 15 minutes:

```toml
[[agent]]
name = "agent"
role = "agent"
mode = "interactive"
options = { heartbeat = 15 }
```

Remove the `heartbeat` option to disable heartbeat for that agent. A heartbeat
is only a ping. If the interactive root agent has something to do, it does it.
If not, it should stay silent.

## Resetting A Local Root Agent

Stop the system first:

```sh
git multiagent stop
```

Then remove only the interactive root agent state if you want a clean interactive root agent:

```sh
rm -rf .git-multiagent/state/agents/agent
```

Start again:

```sh
git multiagent start
```

Do not delete task/job logs or transcripts if you need the run history.

## License

RepoClaw is MIT licensed. See `LICENSE`.

Portions of the Markdown workspace material are derived from OpenClaw, also MIT
licensed. See `THIRD_PARTY_NOTICES.md`.
