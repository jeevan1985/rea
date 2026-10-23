# Working on Jeevan's REA fork

This fork is published at <https://github.com/jeevan1985/rea>. The original
project is <https://github.com/morluto/rea>; its attribution, license and release
identity remain intact. Installing `rea-agents` from npm installs the original
project's published package. Pushing source changes to this fork does not
publish an npm release or update an already installed MCP server.

## Clone and select the publishing destination

```sh
git clone https://github.com/jeevan1985/rea.git
cd rea
git remote add upstream https://github.com/morluto/rea.git
git config remote.upstream.pushurl DISABLED
git config remote.pushDefault origin
git config push.default current
git config pull.rebase false
git config pull.ff true
git config branch.main.rebase false
git config branch.main.pushRemote origin
git fetch upstream main
git branch --set-upstream-to=upstream/main main
```

`origin` publishes to Jeevan's fork. `upstream` fetches the original author's
updates and has a disabled push URL. On `main`, plain `git pull` merges from
`upstream/main`, and plain `git push` publishes to `origin/main`. Merge mode
preserves existing commit history instead of rebasing it. These settings are
local to the checkout; Git does not distribute remote configuration with
commits. Existing checkouts can use `git remote set-url` to correct an existing
remote instead of adding it again. Inspect `git remote -v` before pushing.

## Publish changes

For a focused change, use a branch and open a pull request **against
`jeevan1985/rea`**:

```sh
git switch -c improve-setup
# Edit files and run the relevant checks from CONTRIBUTING.md.
git add <changed-files>
git commit -m "docs: improve local setup instructions"
git push -u origin improve-setup
```

Once a branch tracks `origin`, plain `git push` publishes to the fork. Git still
requires an explicit commit and push; saving a file alone does not publish it.
Do not commit installed engines, binaries, credentials, local configuration,
analysis results, logs, `dist/`, or `node_modules/`.

## Bring in original project updates without losing fork changes

Commit your work or stash it before updating. Then, on `main`:

```sh
git switch main
git pull
# Resolve any merge conflicts, then run the relevant checks.
git push
```

With no divergent changes, the pull fast-forwards. With fork-specific commits,
Git merges both histories. If the same lines changed on both sides, Git pauses
for conflict resolution; inspect the files, preserve the intended behavior,
stage the resolutions and complete the merge before pushing. Do not use
`git reset --hard upstream/main` or force-push as a synchronization method.

This workflow fetches new changes when you run `git pull`; it is not a scheduled
sync or notification service. To preview incoming commits before merging:

```sh
git fetch upstream main
git log --oneline main..upstream/main
git diff --stat main...upstream/main
```

For a larger update that needs review, integrate it on a branch instead:

```sh
git switch main
git switch -c sync-original
git fetch upstream main
git merge upstream/main
# Resolve conflicts and run the relevant checks.
git push -u origin sync-original
```

Open that branch's pull request against the fork. Preserve both histories;
force-pushing is not part of this workflow. See [CONTRIBUTING.md](../CONTRIBUTING.md)
for development and release checks.

If another machine or collaborator has pushed to your fork, also fetch `origin`
and merge `origin/main` before publishing. The default pull on `main` checks the
original author's branch; it does not automatically merge both remotes.
