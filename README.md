# custom-php-repo 8.6 Alpha / Beta / RC

A self-hosted APT repository for pre-release PHP builds (beta/RC), built entirely
on GitHub Actions and published on GitHub Pages — no waiting for upstream
maintainers to publish packages themselves.

Unlike a single monolithic `.deb`, this produces the **same split of packages**
that `deb.sury.org` publishes — `php8.6-cli`, `php8.6-fpm`, `php8.6-common`,
`php8.6-gd`, `php8.6-mysql`, `php8.6-curl`, `php8.6-mbstring`, `php8.6-xml`,
`php8.6-zip`, `php8.6-bcmath`, `php8.6-intl`, `php8.6-ldap`, `php8.6-readline`,
`php8.6-soap`, `php8.6-sqlite3`, `php8.6-bz2`, `php8.6-xsl`, `php8.6-dev`, etc. —
because it starts from the **actual Debian php-team packaging** and adapts it
to the new version, rather than hand-rolling packaging rules from scratch.

## Two workflows, two different frequencies

This repo intentionally has **two separate workflow files**, run for
different reasons:

| File | When to run it | What it does |
|---|---|---|
| `.github/workflows/bootstrap-packaging.yml` | **Rarely.** Once, when setting the repo up, or later if you deliberately want to re-sync from a newer Debian base. | Clones the last *released* PHP series' real Debian packaging from `salsa.debian.org`, mechanically rewrites `8.5` → `8.6` throughout, and commits the result into **this repo** at `packaging/debian/`. |
| `.github/workflows/build-php86.yml` | **Every time** a new beta/RC of PHP drops. | Uses `packaging/debian/` exactly as committed in this repo. Never talks to salsa.debian.org, never mentions "8.5". Builds, tests, and (optionally) publishes the `.deb`s. |

The point of splitting them: once `packaging/debian/` is adopted into this
repo, it's *yours* — tracked in git, editable directly, evolved release to
release the same way Sury and Remi Collet maintain one persistent packaging
tree over time. The day-to-day build never re-derives it from Debian.

## `bootstrap-packaging.yml` — run once

Inputs:

| Input | Default | Meaning |
|---|---|---|
| `base_series` | `8.5` | Last *released* PHP series to copy Debian packaging from |
| `target_series` | `8.6` | Series you're packaging |
| `commit_branch` | `packaging-bootstrap` | Branch the result is pushed to — **review the diff before merging to `main`** |

What it does, step by step:

1. Looks for the Debian packaging branch for `base_series` in
   `https://salsa.debian.org/php-team/php.git` — tries `debian/main/<series>`
   first, then falls back to the older `master-<series>` naming. Fails fast
   with the real branch list if neither exists.
2. Clones that branch (`debian/` directory only needs to exist).
3. Renames every file/path containing `8.5` to `8.6`, and rewrites every
   occurrence of `8.5` to `8.6` inside `debian/`.
4. Replaces `packaging/debian/` in this repo with the rewritten tree.
5. Commits and force-pushes to `commit_branch`, then tells you to open a PR
   from that branch into `main` and merge it.

After that PR is merged, `build-php86.yml` never needs `base_series` again —
it just uses `packaging/debian/` as committed.

## `build-php86.yml` — the day-to-day build

Inputs:

| Input | Default | Meaning |
|---|---|---|
| `php_tag` | `php-8.6.0beta1` | Git tag in `php/php-src` to build |
| `target_series` | `8.6` | PHP series being built — must match `packaging/debian/` (`php8.6-*` packages) |
| `pkg_upstream_version` | `8.6.0~beta1` | Debian-ordered upstream version string (`~` sorts before nothing, so `~beta1` < final) |
| `pkg_revision` | `1` | Debian package revision suffix |
| `publish` | `true` | Publish to your APT repo on GitHub Pages if build + smoke-test pass |

The distro is **not** an input — it's hardcoded (`DISTRO_CODENAME: jammy`,
i.e. Ubuntu 22.04), because that's the only target this pipeline is built
for. `ppa:ondrej/php` is always added as a build-dependency source during
`prepare-and-validate` and `build` — there's no toggle for it. `lintian` is
always run with `--fail-on error` — there's no toggle to relax that either.

Five jobs run in order; a failure at any stage stops the run there:

1. **resolve** — confirms `php_tag` actually exists in `php/php-src`, and
   that `packaging/debian/control` in this repo has `php<target_series>-*`
   packages (i.e. bootstrap was run and merged for this series). Fails in
   seconds with a clear message if either is wrong, instead of failing 20
   minutes into a build.
2. **prepare-and-validate** — installs the packaging toolchain, adds
   `ppa:ondrej/php`, fetches `php-src` at `php_tag` and builds the orig
   tarball, copies `packaging/debian/` in, adds a changelog entry via `dch`,
   then *validates without compiling*: `apt-get build-dep --simulate`
   (catches missing/renamed build-deps), `dpkg-source -b` (catches
   structural mistakes), and `lintian --fail-on error`. Uploads the prepared
   source tree as an artifact.
3. **build** — only if validation passed. Installs real build-dependencies
   with `mk-build-deps` and compiles with `dpkg-buildpackage -us -uc -b`
   directly on the GitHub-hosted runner (this pipeline does **not** use an
   isolated `sbuild`/`mmdebstrap` chroot). Uploads the built `.deb` files.
4. **smoke-test** — only if the build succeeded. Installs the freshly built
   packages in a brand-new, disposable `ubuntu:22.04` container and runs
   `php -v`, `php -m`, `php-fpm -t` (if present). If this repo has a
   `tests/smoke.sh`, it's run here too, before anything can publish — see
   below.
5. **publish** — only if `smoke-test` passed and `publish: true`. Imports
   your GPG signing key from the `APT_GPG_PRIVATE_KEY` secret, checks out
   (or creates) the persistent `apt-repo-state` branch, runs
   `reprepro includedeb` against it, exports the public signing key, writes
   a plain `index.html`, commits, and pushes. **There is no separate
   `testing` component and no manual-approval gate** — a successful,
   smoke-tested run publishes straight to the `main` component that your
   servers track.

```
resolve → prepare-and-validate → build → smoke-test → publish
                                                    (auto, no approval gate)
```

### Why a persistent git branch instead of just deploying to Pages directly

`actions/deploy-pages` replaces the entire site on every run. If the reprepro
repository were rebuilt fresh in the runner's temp filesystem each time,
every run would silently delete every package from every previous run —
`apt upgrade` on your servers would have nothing to see, and old versions
would vanish. Instead, the reprepro repository (its `pool/`, `dists/`, and
`db/`) lives permanently in a dedicated branch, `apt-repo-state`, created
automatically the first time `publish` runs. Every publish run checks that
branch out, runs `reprepro includedeb` against it (which *adds* to it),
commits, and pushes. GitHub Pages then serves that branch directly, so
nothing is ever silently wiped.

One consequence worth knowing: because built `.deb` files are committed to
that branch, its size grows over time. This is by design (it's what makes
"nothing gets deleted" possible) — prune it occasionally once it accumulates
old pre-release builds you don't need anymore:

```bash
git fetch origin apt-repo-state && git checkout apt-repo-state
reprepro -b . remove jammy php8.6 php8.6-cli php8.6-fpm ... # old versions
git add -A && git commit -m "prune old builds" && git push
```

## What this does NOT do

- **No PECL extensions.** `imagick`, `redis`, `igbinary`, etc. are not part
  of Debian's `php-team` packaging and are not built here. They'd need a
  separate `phpize`/`pecl` build job.
- **No `testing`/`main` split and no manual approval gate.** Every
  successful, smoke-tested run with `publish: true` goes straight to the
  component your servers track. If you want a staging step before
  production, that's not built in here — add it deliberately (e.g. don't
  point production servers at this repo until you've reviewed a run's
  smoke-test logs yourself).
- **No isolated build chroot.** `dpkg-buildpackage` runs directly on the
  GitHub-hosted runner, not inside a clean `sbuild`/`mmdebstrap` chroot. It's
  still a fresh, disposable VM per run, but it's not chroot-isolated from
  whatever else is preinstalled on that runner image.
- **No guarantee of a clean first build.** Porting Debian packaging across a
  version Debian hasn't packaged yet (`bootstrap-packaging.yml`) is a
  best-effort mechanical rewrite. `prepare-and-validate` is designed to
  catch most breakage before the expensive compile, but it can't catch
  everything — e.g. a genuinely new build-dependency Debian introduced that
  Ubuntu's archives don't have at all.
- **Single architecture/distro.** `amd64` / `jammy` (Ubuntu 22.04) only,
  hardcoded. A different target means editing the workflow, not just an
  input.

## Testing your own products before anything is published

You almost certainly have applications you don't want a bad PHP build to
break. The `smoke-test` job runs generic checks (`php -v`, loaded modules,
`php-fpm -t`), but it can't know what *your* products need. Add a
`tests/smoke.sh` to this repository and it will be run automatically, inside
the same clean container, after the new packages are installed and before
`publish` runs:

```bash
#!/usr/bin/env bash
set -euo pipefail
php8.6 -r "require 'vendor/autoload.php'; // your own sanity checks"
composer --working-dir=/path/to/your/app check-platform-reqs
```

Whatever exits non-zero here fails the whole run and blocks publishing.

## One-time setup

### 1. Create the repository

Create a new, empty GitHub repository (see the step-by-step guide below for
exact field values), then add both workflow files at:

```
.github/workflows/bootstrap-packaging.yml
.github/workflows/build-php86.yml
```

(copy them in verbatim — do not merge them into one file, they're meant to
stay separate)

### 2. Generate a dedicated GPG signing key

Do this once, locally — not in CI. Use a key **without a passphrase** so CI
can sign non-interactively (it's a dedicated repo-signing key, not your
personal key):

```bash
gpg --batch --pinentry-mode loopback --passphrase '' \
  --quick-generate-key "Custom PHP Repo <ci@example.com>" rsa4096 sign 0

# find the key you just made
gpg --list-secret-keys --keyid-format=long

# export the PRIVATE key, base64-encode it (this is what goes into the secret)
gpg --export-secret-keys --armor <KEYID> | base64 -w0 > private-key.b64
```

### 3. Add the signing key as a repository secret

GitHub repo → **Settings → Secrets and variables → Actions → New repository secret**

- Name: `APT_GPG_PRIVATE_KEY`
- Value: contents of `private-key.b64`

Then delete `private-key.b64` locally (or keep it somewhere safe — it's the
only copy outside GitHub's secret store).

### 4. Run `bootstrap-packaging.yml` once

**Actions → Bootstrap packaging (run once, not per-build) → Run workflow**,
leave the defaults (`base_series: 8.5`, `target_series: 8.6`,
`commit_branch: packaging-bootstrap`) unless you have a reason to change
them. When it finishes, open a PR from `packaging-bootstrap` into `main`,
review the diff, and merge it. `packaging/debian/` now exists in `main`.

### 5. Enable GitHub Pages, serving the `apt-repo-state` branch

`apt-repo-state` is created automatically the first time `build-php86.yml`
runs with `publish: true` — **run it once first** before doing this step.

GitHub repo → **Settings → Pages → Build and deployment → Source: Deploy
from a branch**, then set:

- Branch: `apt-repo-state`
- Folder: `/ (root)`

This is deliberately the classic "serve a branch" mode, not the
Actions-based `deploy-pages` flow — it's what lets the workflow update the
site with a plain `git push` instead of replacing the whole site on every
run.

### 6. Run the build

**Actions → Build PHP 8.6 APT Repository (Ubuntu 22.04 / jammy) → Run
workflow**, with inputs:

| Input | Example | Meaning |
|---|---|---|
| `php_tag` | `php-8.6.0beta1` | Git tag in `php/php-src` to build |
| `target_series` | `8.6` | Series being built — must match `packaging/debian/` |
| `pkg_upstream_version` | `8.6.0~beta1` | Debian-ordered upstream version string |
| `pkg_revision` | `1` | Debian package revision |
| `publish` | `true` | Publish if build + smoke-test pass |

Bump `php_tag` / `pkg_upstream_version` each time PHP cuts a new beta/RC
(`php-8.6.0beta2`, `php-8.6.0rc1`, ...) and re-run.

## Using the repository on your servers

```bash
curl -fsSL https://<you>.github.io/<repo>/apt-repo-signing-key.gpg \
  -o /usr/share/keyrings/custom-php.gpg

echo "deb [signed-by=/usr/share/keyrings/custom-php.gpg] https://<you>.github.io/<repo> jammy main" \
  | sudo tee /etc/apt/sources.list.d/custom-php.list

sudo apt update
sudo apt install php8.6-cli php8.6-fpm php8.6-common php8.6-mysql php8.6-gd
```

Because it's a real APT repo, `apt upgrade` picks up new builds automatically
whenever you re-run `build-php86.yml` with a newer tag and `publish: true`.

## Safety note

This installs **pre-release, third-party-built PHP packages, unofficially
signed by a key you generated yourself**. There is no `testing`/`main` split
and no manual approval gate in this version of the pipeline — a green run
with `publish: true` reaches the component your servers track immediately.
`smoke-test` (plus your own `tests/smoke.sh`) is the only safety net before
that happens, so keep it meaningful, and be deliberate about which servers
point at this repo and when you re-run the workflow.
