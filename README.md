# slop-conf

Configuration as code for [slop](https://github.com/WikiOasis/slop), WikiOasis's deployment tool. slop checks this
repository out as its `config/` submodule.

slop pulls this repository before every build and deploy, and commits and pushes it after every **successful**
deploy. So you can change things either way:

- run slop commands on the deploy master (`slop repo pin`, `slop patch add`, ...), which edit these files and get
  committed by the next successful deploy, or
- edit the files in your IDE, open a pull request, merge it, and deploy.

## Layout

| Path                              | What it is                                                                    |
| --------------------------------- | ----------------------------------------------------------------------------- |
| `slop.yaml`                       | Global settings and projects: where things are built, how they deploy         |
| `inventory.yaml`                  | Hosts, roles (matched by hostname glob), batch sizes, soak times and services |
| `mediawiki/repos-<v>.yaml`        | Extensions and skins for MediaWiki `<v>`, plus `core:` pins                   |
| `mediawiki/patches-<v>.yaml`      | Patches applied on top of checkouts after every sync                          |
| `mediawiki/patches/*.patch`       | Patch files, including conflict resolutions saved by slop                     |
| `mediawiki/scripts/`              | Copied to `<farm>/scripts/`                                                   |
| `state/<project>.yaml`            | **The statefile**: exactly what is deployed. Written by slop, never by hand   |
| `state/<project>/composer-*.lock` | The deployed `composer.lock` per MediaWiki version                            |
| `locks/<project>.yaml`            | Deploy locks from `slop lock`; `_all.yaml` locks everything. Written by slop  |
| `slop.local.yaml`                 | Local overrides for development (gitignored; see `slop.local.example.yaml`)   |

## Repos

```yaml
extensions:
  - path: extensions/CheckUser
    url: https://github.com/wikimedia/mediawiki-extensions-CheckUser
    branch: _version_
    commit: 3f2a1bc0d9e8...
    task: T123
```

`branch` defaults to `_version_` (the release branch, `REL1_46`). `commit` pins the checkout to an exact commit
instead of the branch tip. `task` records why. Core is pinned with a top-level `core: {commit: ...}`.

## Patches

```yaml
patches:
  - name: gerrit-1234567
    path: extensions/Echo
    gerrit: 1234567
    patchset: 3
    project: mediawiki/extensions/Echo
    commit: 9c1e...
    description: Fix notification badge
    task: T456
  - name: github-WikiOasisMagic-42
    path: extensions/WikiOasisMagic
    github: WikiOasis/WikiOasisMagic#42
    commit: 7d2f...
    commits: 2
  - name: fix-uploads
    path: .
    patch: fix-uploads.patch
    description: Raise the upload limit check
  - name: commit-ab12cd34ef
    path: skins/Citizen
    commit: ab12cd34ef...
    url: https://github.com/StarCitizenTools/mediawiki-skins-Citizen
```

`path` is `.` for core. Gerrit and GitHub patches are pinned to a patch set or commit when added
(`slop patch add`) and only move with `slop patch update`. If a patch stops applying, an interactive build lets you
resolve the conflict and saves the result as a `patch:` file next to the original source, which then takes
precedence. Tasks on patches are logged to the SAL (and so to Phorge) when the patch is first deployed.

## The statefile

`state/<project>.yaml` records the last successful deploy: who, when, why, the Phorge tasks, and for every checkout
the upstream commit (`base`), the resulting commit (`head`) and the patches applied. Together with the files at the
same commit, it is enough to rebuild that deploy exactly, which is how `slop rollback` and automatic rollback work.
Every deploy is one commit here, so `git log state/mediawiki.yaml` (or `slop state --history`) is the deploy history.

## Locks

`slop lock -m 'why'` writes `locks/<project>.yaml`, commits and pushes it; if the push fails the lock is undone.
Every deploy and rollback, wherever it is run, pulls this repository first and refuses to go ahead if the pull fails
or someone else holds a lock, so a host with a stale checkout cannot miss a lock. `slop unlock` removes it
(`--force` breaks someone else's), and `--for 2h` makes one expire.

## Checks

CI runs `slop config validate` and `prettier --check`. Format locally with `npx prettier --write .`.
