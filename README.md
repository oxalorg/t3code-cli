# t3code-cli

Create [T3 Code](https://github.com/pingdotgg/t3code) threads, start agent
turns in them, and read their replies from the command line. It comes with a
Claude Code skill, so an agent can hand work off to new T3 threads and
collect the results.

Unofficial, and not affiliated with T3. It uses T3's internal API (tested on
T3 Code v0.0.44), so a T3 update can break it.

## Why I built this

I run most of my coding agents inside T3 Code, and I wanted one agent to be
able to start others: "spin up a thread in that repo and fix X", or fan a
task out across a few projects, then read back what each one did.

T3 doesn't offer that to agents. The `t3-code` MCP server that T3 gives its
sessions only covers the current thread's browser, devices and pull requests.
Adding T3's `/mcp` endpoint to Claude Code by hand doesn't work either:
`claude mcp login` fails, because the endpoint has no OAuth flow and expects a
token that T3 creates for its own sessions.

The T3 web app, though, does everything through a WebSocket API on the local
server. `t3code-cli` sends the same orchestration commands the web app sends,
so threads it creates are ordinary T3 threads: they show up in the sidebar,
and you can open and continue them as usual.

## Requirements

- T3 Code running on the same machine (for example the background service
  from `t3 service`). It listens on port 3773 by default; set `T3_PORT` if
  yours differs.
- The `t3` CLI on your `PATH`. Each run issues a 10-minute bearer token with
  `t3 auth session issue` and revokes it on exit.
- Node.js 22 or newer.

## Install

```sh
git clone https://github.com/oxalorg/t3code-cli ~/projects/t3code-cli
mkdir -p ~/.local/bin
ln -s ~/projects/t3code-cli/t3code-cli ~/.local/bin/t3code-cli

# Optional: the Claude Code skill
mkdir -p ~/.claude/skills
ln -s ~/projects/t3code-cli/skills/t3code-cli ~/.claude/skills/t3code-cli
```

Check it can reach your server:

```sh
t3code-cli projects
```

### Let your agent install it

Paste this into Claude Code (or another coding agent) running on the machine
where T3 Code runs:

```text
Install t3code-cli from https://github.com/oxalorg/t3code-cli for me.

1. Check that the requirements are met: `node --version` is 22 or newer,
   `t3 --version` works, and a T3 Code server is running locally (port 3773
   unless I've set T3_PORT). If anything is missing, stop and tell me.
2. Clone the repo to ~/projects/t3code-cli (ask me first if that folder
   already exists).
3. Symlink ~/projects/t3code-cli/t3code-cli to ~/.local/bin/t3code-cli, and
   tell me if ~/.local/bin is not on my PATH.
4. Symlink ~/projects/t3code-cli/skills/t3code-cli to
   ~/.claude/skills/t3code-cli so Claude Code picks up the skill.
5. Run `t3code-cli projects` and show me the output to confirm it can reach
   the server. Do not create any threads.
```

## Usage

```sh
t3code-cli projects                          # id, title, workspace root
t3code-cli project add <path> [--title T]    # register a folder as a T3 project
t3code-cli threads [--project P] [--all]     # unsettled threads; --all adds settled
t3code-cli new [--project P] [--title T] [--model M] [--provider ID] \
               [--runtime MODE] [--plan] [--worktree [--base B] [--branch NAME]] \
               "<prompt>"                    # prints the new thread id
t3code-cli send <id> [--plan] "<prompt>"     # follow-up turn
t3code-cli wait <id> [--timeout SECONDS]     # block until the turn ends
t3code-cli status <id>                       # JSON summary
t3code-cli read <id> [--all]                 # latest reply, or the whole transcript
t3code-cli archive <id>
```

`P` is a project id, title or path, and defaults to the current directory. A
prompt of `-` is read from stdin.

Delegate a task and collect the result:

```sh
id=$(t3code-cli new --project ~/code/myapp "Fix the flaky login test")
t3code-cli wait "$id"
t3code-cli read "$id"
```

With the skill installed, you can just ask Claude Code: "start a T3 thread in
myapp that fixes the flaky login test, and tell me what it did".

See [`skills/t3code-cli/SKILL.md`](skills/t3code-cli/SKILL.md) for every
option and its defaults.

## Notes

- Use `t3code-cli project add`, not `t3 project add`, while the server is
  running. `t3 project add` writes the database directly, and the running
  server then rejects new threads for that project.
- With `--worktree`, T3 creates the worktree under `~/.t3/worktrees/` on a
  new `t3code/…` branch.
- Without `--model` or `--provider`, a new thread uses the project's default
  model, else the model of the project's most recent thread.
