# RepoClaw

RepoClaw is an OpenClaw-like resident agent living in a git repository.

It gives a repo its own persistent AI presence: a console you can talk to,
memory files, heartbeat behavior, task history, and a small worker team for
planning and delegated work.

RepoClaw is implemented with GitAgents. GitAgents provides the process model,
tasks, jobs, roles, state directory, and console plumbing. RepoClaw is the
resident agent layer that lives inside a repository and learns that repository
over time.

## One Repo, One Claw

You can have a separate RepoClaw resident agent for each git repository.

That is the intended use. Each project can have its own agent identity, memory,
notes, task history, and local habits. Different repositories should not share
the same working memory unless you explicitly want that.

A RepoClaw instance belongs to the repository it serves. It reads that repo's
root `AGENTS.md`, keeps runtime state under that repo's `.git-agents/state/`,
and can grow project-specific files such as `MEMORY.md`, `USER.md`,
`IDENTITY.md`, and daily notes.

This repository is the reusable base. Concrete instances should live in their
own repositories.

## What You Get

RepoClaw gives a git repo:

- an interactive console agent you can talk to from the terminal;
- first-run bootstrap instructions for shaping the agent identity;
- persistent Markdown memory and notes;
- heartbeat pings so the console can notice work without constant prompting;
- a planner role for organizing delegated work;
- standing subagents for bounded background investigation or checks;
- task/job based communication through GitAgents;
- restart recovery so the console can read its transcript and resume context.

It is useful when you want a repository to have an ongoing agent presence rather
than only ad hoc chat sessions.

## Basic Use

Start the workspace:

```sh
git agents start
```

Talk to the console:

```sh
git agents prompt "hello"
```

Check status:

```sh
git agents status
```

Stop it:

```sh
git agents stop
```

Restart an already-running workspace:

```sh
git agents start --restart
```

## First Run

On a fresh instance, the console should follow `BOOTSTRAP.md`.

That bootstraps the workspace identity: who the agent is, who the human is, what
tone to use, and what should be written into memory. After bootstrap, the
instance can commit its own identity and memory files in its own repo.

RepoClaw itself should stay generic. Do not commit one project's identity or
memory back into this base unless you want that behavior to become the default
for every future instance.

## Using It With Another Git Repo

Create a RepoClaw instance for a project by copying or cloning this base into
the project workspace you want the agent to serve. Then start GitAgents from
that repo root.

Typical flow:

1. Create or clone a repo-specific RepoClaw workspace.
2. Start it with `git agents start`.
3. Talk to it with `git agents prompt`.
4. Let bootstrap create the repo-specific identity and memory.
5. Commit those repo-specific files in that instance repo.

The important rule is separation: each target repository can have its own
RepoClaw, with its own `.git-agents/state/`, memory, and habits.

## Files You Edit

The main user-facing files are:

- `AGENTS.md`: the root instruction entry point;
- `BOOTSTRAP.md`: first-run identity setup;
- `SOUL.md`: default identity and continuity guidance;
- `HEARTBEAT.md`: heartbeat notes;
- `TOOLS.md`: local tool notes;
- `SPAWN.md`: how delegation maps to GitAgents tasks and jobs.

Roles live under `.git-agents/roles/`.

Most day-to-day customization happens in the root Markdown files and in
instance-specific memory files.

## Runtime State

Runtime state is generated under:

```text
.git-agents/state/
```

That directory contains pids, transcripts, job state, logs, and Pi session data.
It is intentionally ignored by git.

Useful debug files include:

```text
.git-agents/state/agents/console/transcript.log
.git-agents/state/agents/console/error.log
.git-agents/state/logs/supervisor.log
.git-agents/state/logs/heartbeat.log
```

Do not commit `.git-agents/state/`.

## Heartbeats

By default, `git agents start` launches a heartbeat that pings the console every
15 minutes.

Change the interval:

```sh
git agents start --heartbeat 5
```

Disable heartbeat:

```sh
git agents start --no-heartbeat
```

A heartbeat is only a ping. If the console has something to do, it does it. If
not, it should stay silent.

## Resetting A Local Console

Stop the system first:

```sh
git agents stop
```

Then remove only the console state if you want a clean console:

```sh
rm -rf .git-agents/state/agents/console
```

Start again:

```sh
git agents start
```

Do not delete task/job logs or transcripts if you need the run history.
