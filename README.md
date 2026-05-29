# RepoClaw

RepoClaw is an OpenClaw-like resident agent living in a git repository.

It gives a repo its own persistent AI presence: an interactive root agent you can talk to,
memory files, heartbeat behavior, task history, and a small worker team for
planning and delegated work.

RepoClaw runs on the `multiagent` command. The runtime provides processes,
tasks, jobs, roles, state, and interactive-agent plumbing; RepoClaw is the
resident agent layer that lives inside a repository and learns that repository
over time.

## One Repo, One Claw

You can have a separate RepoClaw resident agent for each git repository.

That is the intended use. Each project can have its own agent identity, memory,
notes, task history, and local habits. Different repositories should not share
the same working memory unless you explicitly want that.

A RepoClaw instance belongs to the repository it serves. It reads that repo's
root `AGENTS.md`, keeps repository config in `.multiagent/`, stores runtime
state in the user-level `~/.multiagent/state/<instance-id>/` tree, and can
grow project-specific files such as `MEMORY.md`, `USER.md`, `IDENTITY.md`,
and daily notes.

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
- task/job based communication through MULTIAGENT;
- restart recovery so the interactive root agent can read its transcript and resume context.

It is useful when you want a repository to have an ongoing agent presence rather
than only ad hoc chat sessions.

## Basic Use

Recommended: start RepoClaw in Docker and mount only the directories it should
inspect or edit:

```sh
multiagent docker start "$PWD" --mount /path/to/project:rw
```

Use one `--mount <directory>:ro` or `--mount <directory>:rw` flag for each
external directory RepoClaw needs. This keeps RepoClaw away from the full host
filesystem by default.

For a trusted host-local run, start the workspace directly:

```sh
multiagent local start
```

Talk to the interactive root agent:

```sh
multiagent local prompt agent "hello"
```

Check status:

```sh
multiagent local status
```

Stop it:

```sh
multiagent local stop
```

Restart an already-running workspace:

```sh
multiagent local restart
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
the project workspace you want the agent to serve. Then start that RepoClaw
workspace with `multiagent`, preferably through Docker with explicit mounts.

Typical flow:

1. Create or clone a repo-specific RepoClaw workspace.
2. Start it with `multiagent docker start <repoclaw-dir> --mount <project-dir>:rw`.
3. Talk to it with `multiagent local prompt agent "hello"`.
4. Let bootstrap create the repo-specific identity and memory.
5. Commit those repo-specific files in that instance repo.

The important rule is separation: each target repository can have its own
RepoClaw, with its own `.multiagent/team.toml`, user-level runtime state,
memory, and habits.

## Files You Edit

The main user-facing files are:

- `AGENTS.md`: the root instruction entry point;
- `BOOTSTRAP.md`: first-run identity setup;
- `SOUL.md`: default identity and continuity guidance;
- `HEARTBEAT.md`: heartbeat notes;
- `TOOLS.md`: local tool notes;
- `SPAWN.md`: how delegation maps to MULTIAGENT tasks and jobs.

Roles live under `.multiagent/roles/`.

Most day-to-day customization happens in the root Markdown files and in
instance-specific memory files.

## Runtime State

Runtime state is generated under the MULTIAGENT registry, not inside the repo:

```text
~/.multiagent/state/<instance-id>/
```

That directory contains pids, transcripts, job state, logs, and Pi session data.
The repo should track `.multiagent/team.toml` and role overrides, not runtime
state.

Useful debug files include:

```text
~/.multiagent/state/<instance-id>/agents/agent/transcript.log
~/.multiagent/state/<instance-id>/agents/agent/error.log
~/.multiagent/state/<instance-id>/logs/supervisor.log
~/.multiagent/state/<instance-id>/logs/heartbeat.log
```

Do not create or commit `.multiagent/state/`; if it appears, remove it.

## Heartbeats

Heartbeat is configured per interactive agent in `.multiagent/team.toml`.
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
multiagent local stop
```

Then reset only the interactive root agent if you want a clean interactive root agent:

```sh
multiagent local agents reset agent --force
```

Start again:

```sh
multiagent local start
```

Do not delete task/job logs or transcripts if you need the run history.

## License

RepoClaw is MIT licensed. See `LICENSE`.

Portions of the Markdown workspace material are derived from OpenClaw, also MIT
licensed. See `THIRD_PARTY_NOTICES.md`.
