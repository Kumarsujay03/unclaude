# unclaude

Remove Claude / AI attribution from a GitHub repo's commit history and make `@claude` disappear from the **Contributors** graph — in one command. A single Bash script, no Python package to install.

## Why @claude shows up

Claude Code appends a trailer to commit messages by default:

```
Co-Authored-By: Claude <noreply@anthropic.com>
```

GitHub treats that trailer as a co-author, so `@claude` appears under **Insights → Contributors**. PRs opened from the CLI can also carry a `🤖 Generated with [Claude Code]` line.

Rewriting history is not enough on its own: GitHub caches the contributors graph, so after a force-push the old count can stick around. `unclaude` cleans the history **and** flushes that cache.

## What it does

1. Bare-clones the repo to a temp dir and counts commits carrying a Claude line.
2. Writes a restorable backup bundle to `~/.unclaude-backups/`.
3. Rewrites commit and tag messages with `git filter-repo`, removing only
   `Co-Authored-By: …claude/anthropic…` and `Generated with Claude Code` lines. Human co-authors are kept.
4. Verifies nothing is left, then force-pushes all branches and tags.
5. Flushes GitHub's contributor cache: renames the default branch away and back via the API, pushes an empty `chore: refresh GitHub contributors` commit, and waits for the graph to rebuild, printing who it lists.

If the history is already clean, it skips straight to step 5.

## Requirements

- `bash`, `git`
- [`git-filter-repo`](https://github.com/newren/git-filter-repo) — `pipx install git-filter-repo`, `brew install git-filter-repo`, or `sudo apt install git-filter-repo`
- [`gh`](https://cli.github.com/) (GitHub CLI), logged in with `gh auth login`, with admin rights on the repo (needed to rename the default branch)
- `python3` (only for `unclaude prevent`)

## Install

```bash
git clone https://github.com/Kumarsujay03/unclaude.git
mkdir -p ~/.local/bin && ln -s "$PWD/unclaude/unclaude" ~/.local/bin/unclaude
```

Or just download the script:

```bash
curl -fsSLo ~/.local/bin/unclaude https://raw.githubusercontent.com/Kumarsujay03/unclaude/main/unclaude && chmod +x ~/.local/bin/unclaude
```

## Usage

```bash
unclaude OWNER/REPO --dry-run     # show what would change, touch nothing
unclaude OWNER/REPO               # clean + force-push + flush graph (asks you to type 'yes')
unclaude https://github.com/OWNER/REPO -y   # no confirmation
unclaude OWNER/REPO --no-refresh  # clean + push only, skip the branch rename and empty commit
unclaude OWNER/REPO --no-backup   # skip the backup bundle (not recommended)
unclaude prevent                  # stop Claude Code adding attribution in future
```

`TARGET` can be `OWNER/REPO`, an `https://github.com/...` URL, a `git@github.com:...` URL, or any other git URL/path (the cache flush is GitHub-only and is skipped for non-GitHub targets).

Example:

```
$ unclaude you/my-app
  cloning you/my-app…

  repo     : you/my-app
  traces   : 3 commit(s) with a Claude co-author/generated line
  plan     : rewrite history, force-push branches + tags, refresh graph

  Continue? type 'yes': yes
  backup   : /home/you/.unclaude-backups/my-app-20261013-124126.bundle
  rewriting history…
  force-pushing branches and tags…
  flushing contributor cache (rename main → main-unclaude-tmp → main)…
  pushing empty refresh commit to main…
  waiting for GitHub to rebuild the graph…
  contributors now: you

  done.
```

## Prevent it happening again

```bash
unclaude prevent
```

Sets `"attribution": {"commit": "", "pr": ""}` and `"includeCoAuthoredBy": false` in `~/.claude/settings.json`.

## Restore from backup

```bash
git clone ~/.unclaude-backups/<name>.bundle restored
```

## Caveats — read before running

- **History is rewritten.** Every affected commit gets a new SHA. Collaborators must re-clone or `git fetch && git reset --hard origin/<branch>`; tags need `git fetch --tags --force`.
- **Branch protection** that blocks force-pushes must be relaxed temporarily.
- **Default branch rename** shows collaborators GitHub's "default branch renamed" notice. Use `--no-refresh` if that's a problem.
- **Empty refresh commit** stays on the default branch. It's harmless; drop it later if you like.
- **Claude as author** (not co-author) is reported but not rewritten — use `git filter-repo --mailmap` for that.
- **Closed pull requests** keep old commits in `refs/pull/N/head`, which only GitHub Support can purge.
- **Graph lag:** even after the flush, give GitHub a few minutes and recheck in a private window.

## Verify independently

```bash
git clone --mirror https://github.com/OWNER/REPO.git /tmp/check
git -C /tmp/check log --all --format='%B' | grep -ciE 'co-authored-by:.*(claude|anthropic)'   # should print 0
```

## Credits

Inspired by [declaude](https://github.com/ediiloupatty/declaude), rewritten from scratch as a single dependency-light shell script.

## License

[MIT](LICENSE)
