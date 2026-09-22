# Git workflow notes

Git 2.43 on Linux/macOS. I switched from `git checkout` to `git switch`/`git
restore` about two years ago and never went back. `checkout` does too many
things and the flags are a minefield.

## Branching

```bash
git switch -c fix/login-timeout      # create + switch (replaces checkout -b)
git switch main                      # switch to existing
git switch -                         # back to previous branch (yes, really)
```

Push a new branch and set upstream in one go:

```bash
git push -u origin fix/login-timeout
```

After that, plain `git push` works. Forgetting `-u` is why you get
`fatal: The current branch has no upstream branch`.

## Undo recipes (the ones I actually need)

| Situation | Command |
|-----------|---------|
| Unstage a file, keep changes | `git restore --staged file.py` |
| Discard working changes to a file | `git restore file.py` |
| Amend last commit message | `git commit --amend -m "new msg"` |
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` |
| Undo last commit, keep changes unstaged | `git reset HEAD~1` |
| Undo last commit AND discard changes | `git reset --hard HEAD~1` |
| Undo a pushed commit | `git revert <sha>` |

> **gotcha**: `git reset --hard` deletes uncommitted work with no prompt and no
> trash can. I bind it to nothing on purpose. Use `git stash` first if you're
> not sure. `git stash -u` includes untracked files (which `git stash` alone
> does NOT).

## Rebase vs merge

I rebase local feature branches onto `main` before opening a PR, and merge for
the PR itself. Team rules vary; ask before you rebase shared branches.

```bash
git fetch origin
git rebase origin/main
# resolve conflicts, then:
git rebase --continue
# or bail out entirely:
git rebase --abort
```

> **gotcha**: rebasing a branch that others have pulled rewrites SHAs and gives
> everyone a conflict headache. Only rebase branches nobody else has.

Interactive rebase to clean up commits before a PR:

```bash
git rebase -i origin/main
# pick / squash / reword / drop in the editor
```

I squash "wip" and "fix typo" commits so the history reads like I knew what I
was doing.

## Reflog: the safety net

If you "lost" commits, they're usually still there for ~90 days.

```bash
git reflog
# abc1234 HEAD@{3}: reset: moving to HEAD~1
git switch -c rescue abc1234     # or: git reset --hard abc1234
```

I have recovered a hard-reset branch this way. Reflog is the single most
underused git command.

## Finding things

```bash
git log --oneline --graph --decorate -20
git log -S "def parse_config" --oneline       # commits that added/removed a string
git log --author="me" --since="2 weeks ago"
git blame -L 40,60 src/app.py
git diff --stat HEAD~3
```

`git log -S` (the "pickaxe") is great for "when did this line appear and why".

## Stash

```bash
git stash push -m "wip: refactor auth"     # named stash, please
git stash list
git stash pop                               # apply + drop
git stash apply stash@{2}                   # apply, keep in list
```

> **gotcha**: `git stash pop` can leave you with conflicts if the tree moved.
> `apply` keeps the stash so you can retry. Prefer `apply` when unsure.

## Fetch vs pull

```bash
git fetch origin          # download, don't touch working tree
git pull --rebase origin main   # fetch + rebase local commits on top
```

I set `git config --global pull.rebase true` so a plain `git pull` doesn't
create merge commits out of nowhere. Configure `pull.ff only` if you'd rather it
refuse and make you think.

## Notes

- `git config --global rerere.enabled true` remembers conflict resolutions and
  reapplies them. Weird name, genuinely useful during long rebases.
- `.gitignore` does NOT untrack files already committed. `git rm --cached
  file` then commit.
- Line ending hell: `git config --global core.autocrlf input` on macOS/Linux.
