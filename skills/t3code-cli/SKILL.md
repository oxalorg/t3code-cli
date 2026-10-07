---
name: t3code-cli
description: Create new T3 Code threads and start agent sessions in them with an initial prompt, send follow-ups, wait for them, and read their replies, using the `t3code-cli` CLI. Use when the user asks to start, spawn, kick off, or hand off work to a new T3 thread or session (in any project), to run several T3 threads in parallel, to have a monitoring thread spawn one fix-and-PR thread per issue it finds, or to check on or read back another T3 thread.
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
t3code-cli new [--project P] [--key KEY] [--title T] [--model M] [--provider ID] \
               [--runtime MODE] [--plan] [--worktree [--base B] [--branch NAME]] \
               "<prompt>"                    # prints the new thread id on stdout
t3code-cli send <id> [--plan] [--runtime MODE] "<prompt>"  # follow-up; thread must be idle
t3code-cli wait <id> [--timeout 1800]        # blocks until the turn ends; prints state
t3code-cli status <id>                       # JSON: state, model, branch, worktree, turn, PRs
t3code-cli find <key>                        # threads started with --key KEY
t3code-cli read <id> [--all]                 # latest reply (plan + final message), or transcript
t3code-cli archive <id>
```

`threads` and `find` print tab-separated lines: id, project, state, last
update, linked PRs (`#12 open`, `-` if none), title.

- `--key KEY` gives the thread a stable identity, e.g. `myapp#42` for an
  issue. If the project already has a thread with that key, `new` prints the
  existing thread's id (and says so on stderr) instead of creating another.
  The key is stored as a `[t3code-cli key: KEY]` line at the end of the
  first message, since T3 renames thread titles.
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

## One fix thread per issue (monitor → fix → PR)

A monitoring thread (watching logs, CI, or a GitHub issue list) can start one
thread per problem it finds. Each fix thread works in its own worktree and
opens a PR. T3 sessions link the PRs they open to their thread, so the user
can follow every fix and its PR state in the T3 sidebar.

1. Give each problem a stable key, such as the issue reference
   (`owner/repo#42`). File an issue first if there is none; it gives the
   key, and gives the fix thread and the PR something to reference.
2. Start the fix thread:

   ```sh
   t3code-cli new --project myapp --key "owner/repo#42" --worktree - <<'EOF'
   Fix https://github.com/owner/repo/issues/42: <one-line summary>.

   What we saw: <error, log lines, how to reproduce>.
   Where to look: <files or modules, if known>.

   Work on the branch of this thread's worktree. Add or update a test that
   fails before the fix, make it pass, and run the project's checks. Then
   push and open a PR whose description says "Fixes #42". If you cannot fix
   it, say why in the issue and stop.
   EOF
   ```

   Re-running this for the same key is safe: it returns the existing thread.
3. Do not `wait` on fix threads from the monitor; keep monitoring. Check on
   them later with `t3code-cli threads --project myapp` (the PR column shows
   `#57 open` / `merged`) or `t3code-cli status <id>`.
4. Limit how many fix threads run at once (for example 3; count the
   `running` rows of `threads`) and queue the rest.
5. Tell the user which issues got a thread, with the thread titles.
