# git-merge-up

**Update a stack of Git branches with merge, the way `git rebase --update-refs` does it with rebase.**

When you work with stacked branches (`main` → `feat-1` → `feat-2` → `feat-3`), keeping them in sync means merging `main` into `feat-1`, then `feat-1` into `feat-2`, and so on. `git merge-up` does the whole chain in one command, with `--continue` and `--abort` for conflicts, just like rebase.

```
$ git merge-up main
Chain: main -> feat-1 -> feat-2 -> feat-3
==> feat-1  <-  main
==> feat-2  <-  feat-1
==> feat-3  <-  feat-2
Done. You are on feat-3.
```

## Why merge instead of rebase?

Rebase rewrites history, which means force-pushing every branch in the stack and breaking anyone else working on them. Merge keeps history intact: no force-push, and shared branches stay safe. `git merge-up` gives you the convenience of `rebase --update-refs` without rewriting anything.

## Features

- **One command for the whole stack**: merges each branch's parent into it, bottom to top.
- **Auto-detects the stack** from the current branch, or takes the branches explicitly.
- **Conflict handling like rebase**: stops on conflict, then `--continue` or `--abort`.
- **Full abort**: restores every branch to the commit it had before the run.
- **Autostash**: uncommitted changes (including untracked files) are saved and restored on the branch where you started.
- **Safe push**: `--push` only pushes once every merge succeeded.
- **Dry run**: preview the plan before touching anything.
- **Single file, no dependencies**: just Bash and Git.

## Installation

Git runs any executable named `git-<name>` found in your `PATH` as `git <name>`.

```bash
mkdir -p ~/bin
curl -fsSL https://raw.githubusercontent.com/sposmen/git-merge-up/main/git-merge-up -o ~/bin/git-merge-up
chmod +x ~/bin/git-merge-up
```

If `~/bin` is not in your `PATH`, add it:

```bash
# zsh (macOS default)
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc

# bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

Check that it works:

```bash
git merge-up -h
```

> `git merge-up --help` does not work: Git intercepts `--help` on every subcommand and looks for a man page instead. Use `-h`, or call `git-merge-up --help` directly.

Optional short alias:

```bash
git config --global alias.mu merge-up   # then: git mu main
```

> **macOS:** if you downloaded the file through a browser, remove the quarantine flag with
> `xattr -d com.apple.quarantine ~/bin/git-merge-up`.

### Tab completion (optional)

Completes options, the base (any ref, such as `main` or `origin/main`) and the local branches of the stack. Both files hook into the shell's existing Git completion, so that must already work (`git checkout <TAB>` lists your branches).

**Zsh** (the macOS default, including Oh My Zsh):

```bash
mkdir -p ~/.zsh/completions
curl -fsSL https://raw.githubusercontent.com/sposmen/git-merge-up/main/completions/_git-merge-up -o ~/.zsh/completions/_git-merge-up
```

Then add this to `~/.zshrc`, **before** `compinit` runs (with Oh My Zsh: before the `source $ZSH/oh-my-zsh.sh` line):

```zsh
fpath=(~/.zsh/completions $fpath)
```

Optionally, give it a description in the `git <TAB>` command list:

```zsh
zstyle ':completion:*:*:git:*' user-commands merge-up:'update a stack of branches with merge'
```

Restart the shell, or run `rm -f ~/.zcompdump*; exec zsh` if the completion does not show up.

**Bash:**

```bash
mkdir -p ~/.bash_completion.d
curl -fsSL https://raw.githubusercontent.com/sposmen/git-merge-up/main/completions/git-merge-up.bash -o ~/.bash_completion.d/git-merge-up.bash
echo 'source ~/.bash_completion.d/git-merge-up.bash' >> ~/.bashrc
```

The `source` line must run after Git's own completion is loaded.

## Usage

```bash
git merge-up [--push] [--dry-run] <base> [branch1 ... branchN]
git merge-up --continue
git merge-up --abort
```

| Option        | Description                                                              |
|---------------|--------------------------------------------------------------------------|
| `--push`      | Push every branch of the stack once all merges succeeded.                |
| `--dry-run`   | Print the merge plan without changing anything.                          |
| `--continue`  | Resume after resolving conflicts (run `git add` first).                  |
| `--abort`     | Cancel and restore every branch, the original checkout and your changes. |
| `-h`          | Show help (`--help` is intercepted by Git; see Installation).            |

### Examples

```bash
# Auto-detect the stack: stand on the top branch and give the base
git checkout feat-3
git merge-up main

# Explicit branches, processed in the given order
git merge-up main feat-1 feat-2 feat-3

# Preview first
git merge-up --dry-run main

# Merge from the up-to-date remote base and push the results
git fetch
git merge-up --push origin/main
```

> **Tip:** `git merge-up` does not fetch. Use `origin/main` as the base (after `git fetch`) to avoid merging a stale local `main`.

## Resolving conflicts

```
$ git merge-up main
Chain: main -> feat-1 -> feat-2 -> feat-3
==> feat-1  <-  main
==> feat-2  <-  feat-1
CONFLICT (content): Merge conflict in a.txt

Conflict while merging 'feat-1' into 'feat-2'.
Resolve it, run 'git add', then: git merge-up --continue
To cancel everything:            git merge-up --abort
```

1. Fix the conflicted files.
2. `git add` them (you can also commit yourself; it's optional).
3. Run `git merge-up --continue`.

The run resumes at the branch that had the conflict and continues up the stack. If you would rather start over, `git merge-up --abort` puts everything back exactly as it was.

## How it works

### Stack detection

Without explicit branches, the current branch is the top of the stack. A local branch is part of the chain when it:

- is an ancestor of the current branch (it sits below it),
- is not already contained in `<base>` (it still has work of its own),
- does not point at the same commit as the current branch (for example, a backup branch; these are skipped with a notice).

Branches are ordered bottom to top by their distance from the top branch. Run `--dry-run` to check the detected chain, or pass the branches explicitly if you keep old branches that point at commits inside your stack.

### Where you end up

- **No local changes:** on the top branch of the stack.
- **With local changes:** back on the branch where you started, with your changes restored there, so work in progress stays on the branch it belongs to.

If the autostash cannot be re-applied cleanly, it is kept in the stash list and the script prints the `git stash apply <sha>` command to recover it.

### State

While a run is in progress, its state is stored in `.git/merge-up/`, the same approach Git uses for rebase (`.git/rebase-merge/`):

| File    | Contents                                                          |
|---------|-------------------------------------------------------------------|
| `meta`  | Base, current step, autostash SHA, starting branch, push flag     |
| `chain` | Branches of the stack, bottom to top                              |
| `orig`  | The commit each branch pointed to before the run (used by abort)  |

The state lives inside `.git`, so it never shows up in `git status` and cannot be committed by accident. It is removed when the run finishes or is aborted.

## Caveats

- **`--abort` resets branches hard.** Each branch in the stack goes back to its original commit, so any extra commits you made on those branches during the conflict are discarded, just like `git rebase --abort`.
- **Push failures.** If a push fails partway through `--push`, earlier branches are already on the remote. Do not use `--abort` at that point; fix the push issue and push the remaining branches manually.
- **Worktrees.** Git does not allow a branch to be checked out in two worktrees at once, so `git merge-up` refuses to start if a branch of the stack is checked out in another worktree, and tells you which one. Free it with `git -C <worktree> switch --detach`, or run `git merge-up` from the worktree that holds the top of the stack. The base can be checked out anywhere. State is kept per worktree, so run `--continue` and `--abort` from the same worktree where you started.
- **Merge history.** Merging up a stack creates a merge commit on each branch. If you need a linear history, a rebase-based workflow fits better.

## Requirements

- Git 2.13 or newer.
- Bash. Tested on Bash 5; written to be compatible with Bash 3.2 (the default on macOS).

## Alternatives

If you need a fuller stacked-branches workflow, these tools also support merge-based syncing:

- [git-machete](https://github.com/VirtusLab/git-machete): `git machete traverse --merge`, with a branch layout file and GitHub/GitLab integration.
- [Git Town](https://github.com/git-town/git-town): `git town sync` with `sync-feature-strategy` set to `merge`.

`git-merge-up` is intentionally smaller: one file, no configuration, and it only does the merge-up.

## License

MIT. See [LICENSE](LICENSE).
