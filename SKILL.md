---
name: fj-skill
description: >-
  Forgejo CLI (fj) wrapper. Forward any fj subcommand, especially repository‑related operations like listing repositories, creating pull requests, fetching issues, merging branches, managing issue labels, and linking issues. This is not useful for the user to invoke this skill directly. Agent should call this skill when interacting with Forgejo repositories via CLI — creating PRs, managing issues, releases, or CI status — or any time you'd reach for `gh` but the repo's remote is a Forgejo or Gitea host rather than GitHub.
metadata:
  source-project: forgejo-contrib/forgejo-cli
---
# Forgejo CLI Wrapper

This skill is a thin wrapper around the official `fj` command‑line client. It forwards any arguments you provide to the underlying `fj` binary and returns the raw output verbatim.

## How to use it
1. Invoke this skill by name: `fj-cli`.
2. Provide the fj subcommand and its flags exactly as you would on the shell. For example:
   ```
   fj-cli repo list
   fj-cli pr create --source feature/foo --target main \
     --title "Add new feature" \
     --description "$(cat /tmp/desc.md)"
   fj-cli issue assign 42 alice bob
   ```
3. The skill will prepend `fj` if you omitted it, execute the command using the `tools.shell` tool, and return only the standard output.
4. No additional commentary or formatting is added.

## Assumptions
- The user has already authenticated to Forgejo using `fj auth login` or `fj auth add-token`.
- The `fj` binary is available on the system path.

## Examples

```
# PRs
fj pr create "Title" --base main --head feature --body "..."   # --base AND --head
fj pr create -A --base main                                    # -A/--autofill from commits
fj pr status 42 --wait                                         # block until checks finish
fj pr merge 42 -M squash -d -t "Title (#42)" -m "Body"         # -M method, -d deletes branch
#   -M methods: merge | rebase | rebase-merge | squash | manual
#   -t/-m set the merge/squash commit title and body; -m with NO argument opens the editor
fj pr review 42 list                                           # list reviews; -c adds inline
#   review comments, -a includes stale/dismissed reviews
fj pr search -s all                                            # -s: open (default) | closed | all
fj pr view 42 body|comment|comments|labels|assignees|diff|files|commits   # subcommand, not a flag

# Issues
fj issue edit 7 labels -a added -r removed                     # -a add, -r/--rm remove
#   ^ succeeds SILENTLY (no output). Don't assume failure; verify with `issue view`.
fj issue view 7                                                # labels shown in plain view
fj issue view 7 body|comment|comments|assignees                # body is the default; there is
#   NO `labels` subcommand (that's pr-only — `issue view 7 labels` errors); `comments`
#   lists every comment, `comment` views one
fj issue close 7 -w "reason"                                   # -w/--with-msg, inline arg required

# Labels — repo-level defs under `fj repo labels [REPO] <subcommand>` (NOT `fj label`)
fj repo labels owner/name view                                 # list existing labels
fj repo labels owner/name create state/approved a3b18a -e      # NAME then COLOR (hex); -e = exclusive
fj repo labels owner/name edit state/approved -c a3b18a -e true # edit by name/ID; on EDIT -e takes true|false
fj repo labels owner/name delete state/approved               # by name or ID
#   -d/--description with NO argument opens $EDITOR — omit it or pass a value.
#   -e is a BARE flag on create but true|false on edit. Exclusivity is per namespace —
#   the `scope` in a `scope/value` name (`scope:value` is NOT treated as a namespace).

# Org-level labels (one shared set across all the org's repos; user repos have no equivalent)
fj org label list|add|edit|rm                                  # define labels once on an organization

# Auth / identity
fj whoami            # current signed-in identity for the active instance
fj auth list         # all instances you're logged into
fj auth login        # opens browser (interactive — not agent-runnable)
fj auth add-token -H <host>   # agent-runnable fallback: token as arg or via stdin
#   Refuses to REPLACE a credential: with one already stored for that host it
#   fails `new key:key for <host> already exists`, leaves the old one in place,
#   and still exits 0 — so a switch that silently did nothing looks like it
#   worked. Verify with `fj whoami`, and log out first to change identity.
fj auth logout <host>         # HOST is POSITIONAL — `-H` errors here
```

## Quick Reference – Help Snippets
The following sections provide the output of `--help` for common subcommands. These are static and will be returned directly when requested.

### `repo list` help
```shell
fj repo list --help
Usage: fj repo list [OPTIONS]

List all repositories you have access to.

Options:
  -H, --host <HOST>
  -C, --cwd <CWD>
  -h, --help
```

### `repo view` help
```shell
fj repo view --help
Usage: fj repo view [OPTIONS] <REPO>

Show information about a repository.

Options:
  -H, --host <HOST>
  -C, --cwd <CWD>
  -h, --help
```

### `pr create` help
```shell
fj pr create --help
Usage: fj pr create [OPTIONS] <REPO> <SOURCE_BRANCH> <TARGET_BRANCH>

Create a new pull request.

Options:
  -H, --host <HOST>
  -R, --remote <REMOTE>
  -C, --cwd <CWD>
  --title <TITLE>
  --description <DESCRIPTION>
  -h, --help
```

### `pr status` help
```shell
fj pr status --help
Usage: fj pr status [OPTIONS] <PR>

Show the mergability and CI status of a pull request.

Options:
  -H, --host <HOST>
  -C, --cwd <CWD>
  -h, --help
```

### `issue list` help
```shell
fj issue list --help
Usage: fj issue list [OPTIONS] <REPO>

List issues for a repository.

Options:
  -H, --host <HOST>
  -R, --remote <REMOTE>
  -C, --cwd <CWD>
  --state <STATE>
  --label <LABELS>
  -h, --help
```

### `issue comment` help
```shell
fj issue comment --help
Usage: fj issue comment [OPTIONS] <ISSUE> <BODY>

Add a comment to an issue.

Options:
  -H, --host <HOST>
  -R, --remote <REMOTE>
  -C, --cwd <CWD>
  -h, --help
```

### `merge` help (via PR merge)
```shell
fj pr merge --help
Usage: fj pr merge [OPTIONS] <PR>

Merge a pull request.

Options:
  -H, --host <HOST>
  -M, --method <METHOD>
  -d, --delete
  -t, --title <TITLE>
  -m, --message [<MESSAGE>]
  -C, --cwd <CWD>
  -h, --help
```

### `repo labels` help
```shell
fj repo labels --help
Usage: fj repo labels [OPTIONS] [REPO] <COMMAND>

Manage a repository's issue labels.

Commands:
  view    Show a repo's labels
  create  Create a new label
  delete  Delete a label
  edit    Edit a label
```

### `issue assign` help
```shell
fj issue assign --help
Usage: fj issue assign [OPTIONS] <ISSUE> [USERS]...

Assign users to an issue.

Options:
  -H, --host <HOST>
  -R, --remote <REMOTE>
  -C, --cwd <CWD>
  -h, --help
```

### `issue link` help
```shell
fj issue link --help
Usage: fj issue link [OPTIONS] <ISSUE> <TARGET_ISSUE>

Create a link between two issues.

Options:
  -H, --host <HOST>
  -R, --remote <REMOTE>
  -C, --cwd <CWD>
  -h, --help
```

