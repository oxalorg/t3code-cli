---
name: t3code-cli
description: Create new T3 Code threads and start agent sessions in them with an initial prompt, send follow-ups, wait for them, and read their replies, using the `t3code-cli` CLI. Use when the user asks to start, spawn, kick off, or hand off work to a new T3 thread or session (in any project), to run several T3 threads in parallel, or to check on or read back another T3 thread.
---

# t3code-cli

`t3code-cli` (`~/.local/bin/t3code-cli`, source `~/projects/t3code-cli/t3code-cli`)
drives the local T3 Code server (port 3773, the `t3code.service`). Each call
issues a 10-minute bearer token with `t3 auth session issue` and revokes it on
exit. New threads show up in the user's T3 sidebar like any other thread.

The `t3-code` MCP tools cannot do any of this; they only cover the current
thread's browser, devices and PRs.

## Commands

```sh
t3code-cli projects                          # id, title, workspace root
t3code-cli project add <path> [--title T]    # prints the project id
t3code-cli threads [--project P] [--all]     # unsettled threads; --all adds settled
t3code-cli new [--project P] [--title T] [--model M] [--provider ID] \
               [--runtime MODE] [--plan] [--worktree [--base B] [--branch NAME]] \
               "<prompt>"                    # prints the new thread id on stdout
t3code-cli send <id> [--plan] [--runtime MODE] "<prompt>"  # follow-up; thread must be idle
t3code-cli wait <id> [--timeout 1800]        # blocks until the turn ends; prints state
t3code-cli status <id>                       # JSON: state, model, branch, worktree, latest turn
t3code-cli read <id> [--all]                 # latest reply (plan + final message), or transcript
t3code-cli archive <id>
```

- `--project` takes a project id, title, or workspace path. Default: the
  project whose root is the current directory. Run `t3code-cli projects` if
  unsure.
- A folder must be a T3 project first. Add it with `t3code-cli project add`,
  never `t3 project add`: that one writes the database behind the running
  server, which then rejects threads for the project.
- Model: `--model` and `--provider` (`claudeAgent` or `codex`) pick it.
  Without them it is the project's default model, else the model of the
  project's most recent thread, which may be Codex. `--provider` alone uses
  the model last run on that provider. Pass both when it matters, e.g.
  `--provider claudeAgent --model claude-opus-5-5`.
- `--runtime` defaults to `full-access`, T3's default.
- `--plan` runs the turn in plan mode; `read` then prints the proposed plan
  followed by the agent's final message.
- `--worktree` creates a git worktree under `~/.t3/worktrees/` on a new
  `t3code/…` branch (T3 may rename it), based on `--base` (default: the
  project's current branch). The project must be a git repo with a commit.
- A prompt of `-` is read from stdin; use that for long prompts:
  `t3code-cli new --project myapp - <<'EOF' ... EOF`.

## How to use it

- Write the initial prompt so the new agent can work alone: it does not see
  this conversation. Include the goal, relevant paths, constraints, and what
  "done" looks like.
- Tell the user which threads you started (title and project); they will see
  them in T3.
- To delegate and collect results: `id=$(t3code-cli new ...)`, then
  `t3code-cli wait "$id"` (run long waits in the background), then
  `t3code-cli read "$id"`.
- `wait` prints `completed` on success; anything else (`error`, `idle`,
  `interrupted`) means check `status` and `read --all`.
- Several `new` calls can run in parallel for independent tasks.
- Do not archive or send follow-ups to the user's own threads unless they ask.
