# Fork and branch guide

This repository is a fork of
[KINGDKAK/PS5-FFPFSC-PRO](https://github.com/KINGDKAK/PS5-FFPFSC-PRO).

## Remotes

| Remote | Repository | Use |
| --- | --- | --- |
| `fork` | `tidewell/PS5-FFPFSC-PRO` | Development |
| `origin` | `KINGDKAK/PS5-FFPFSC-PRO` | Upstream source |

In a new clone, set this up with:

```bash
git clone https://github.com/tidewell/PS5-FFPFSC-PRO.git
cd PS5-FFPFSC-PRO
git remote rename origin fork
git remote add origin https://github.com/KINGDKAK/PS5-FFPFSC-PRO.git
git remote set-url --push origin no_push
git config remote.pushDefault fork
git switch all-fixes
```

## Branches

| Branch | Base | Content |
| --- | --- | --- |
| `main` | Upstream `main` | Copy of upstream |
| `update/dependencies` | `main` | MkPFS 1.1.0, UnRAR 7.23 and their documentation |
| `fix/linux-gui` | `main` | Linux interface fixes and their documentation |
| `all-fixes` | `main` | Every fix combined, current and future |

`all-fixes` is the branch to run and to continue working on. The other two are
independent, so each one can be reviewed, rebuilt or reused on its own:

```text
main ─┬─ update/dependencies ─┬─ all-fixes
      └─ fix/linux-gui ───────┘
```

## Adding a fix

Small fix, committed straight to the combined branch:

```bash
git switch all-fixes
git add <files>
git commit
git push fork all-fixes
```

Larger or self-contained fix, kept on a topic branch:

```bash
git switch -c fix/short-name main
git add <files>
git commit
git push -u fork fix/short-name
git switch all-fixes
git merge --no-ff fix/short-name
git push fork all-fixes
```

For new work on dependencies or the Linux interface, commit to the matching
branch and merge that branch into `all-fixes` the same way.

## Getting upstream changes

```bash
git fetch origin
git switch main
git merge --ff-only origin/main
git push fork main
git switch all-fixes
git merge main
git push fork all-fixes
```

Local `main` tracks the upstream branch. Because `remote.pushDefault` is set, a
plain `git push` on it also goes to the fork.

When upstream changes `backend/mkpfs` or `backend/unrar/src`, keep the newer
version of each dependency and follow
[DEPENDENCIES.md](DEPENDENCIES.md) to replace the files as a whole, rather than
resolving conflicts line by line inside vendored code.

## Behavior inherited from upstream

The fork keeps these upstream settings:

- The update checker reads releases from `KINGDKAK/PS5-FFPFSC-PRO`, so it can
  offer an upstream release that does not contain the fixes from this fork.
- Compatibility reports go to the upstream community database.
- `backend/.github/workflows/build-gui.yml` is not at the repository root, so
  GitHub Actions does not run it.

## Related documents

- [DEPENDENCIES.md](DEPENDENCIES.md): dependency versions and update procedure.
- [LINUX.md](LINUX.md): Linux setup and interface fixes.
