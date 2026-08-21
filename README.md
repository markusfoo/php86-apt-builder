# PHP 8.6 Debian Packages (Ubuntu 22.04 / jammy / amd64)

Pre-built `.deb` packages for PHP 8.6 beta/RC releases, published as
**GitHub Releases** with all assets attached. No APT repository setup
required -- just download the `.deb` files and install.

Built from the **real Debian php-team packaging**, adapted for PHP 8.6.
Produces the same split of packages that `deb.sury.org` publishes
(`php8.6-cli`, `php8.6-fpm`, `php8.6-common`, `php8.6-gd`, `php8.6-mysql`,
`php8.6-curl`, `php8.6-mbstring`, `php8.6-xml`, `php8.6-zip`, `php8.6-bcmath`,
`php8.6-intl`, `php8.6-ldap`, `php8.6-readline`, `php8.6-soap`,
`php8.6-sqlite3`, `php8.6-bz2`, `php8.6-xsl`, `php8.6-dev`, etc.).

**PECL extensions** are built automatically as part of every release:
xdebug, redis, igbinary, msgpack, apcu, mongodb, imagick (patched for PHP 8.6),
memcached. Extensions that fail to compile are skipped with a warning.

## Quick start

```bash
# Download all .deb files from the latest GitHub Release
# (or pick individual packages you need)

sudo dpkg -i php*.deb
sudo apt-get -f install

php8.6 -v
php8.6 -m
```

## Workflow overview

```
resolve -> prepare-and-validate -> build -> smoke-test --+-> publish
                                           build-pecl  --+
```

| Job | What it does |
|---|---|
| **resolve** | Confirms the php-src tag exists and packaging matches the target series |
| **prepare-and-validate** | Fetches source, validates packaging without compiling |
| **build** | Compiles all PHP 8.6 `.deb` packages |
| **smoke-test** | Installs in a clean container and runs `php -v`, `php -m`, `php-fpm -t` |
| **build-pecl** | Builds PECL extensions against the compiled PHP (runs in parallel with smoke-test) |
| **publish** | Uploads new `.deb` files to the GitHub Release (smart upsert -- never deletes existing assets) |

### PECL extensions

These are built automatically in the `build-pecl` job. To add or remove
extensions, edit the `EXTENSIONS` variable in the workflow:

```yaml
EXTENSIONS="xdebug redis igbinary msgpack apcu mongodb imagick memcached"
```

Imagick includes a build-time patch for PHP 8.6 API compatibility
(`zend_is_callable` -> `zend_is_callable_ex`). If other extensions need
patches for PHP 8.6, add them in the same `case` block.

### Smart release management

The `publish` job does **not** delete and recreate releases. Instead:

- If a release for this version tag already exists, only **new** `.deb`
  files are uploaded (existing assets are kept).
- If new PECL extensions are added, the release description is updated
  with a dated entry listing the extension name and version.
- Beta/RC/alpha versions are automatically marked as pre-release.

This means you can re-run the workflow and it will only add what's missing.

## `build-php86.yml` -- the day-to-day build

Inputs:

| Input | Default | Meaning |
|---|---|---|
| `php_tag` | `php-8.6.0beta1` | Git tag in `php/php-src` to build |
| `target_series` | `8.6` | PHP series -- must match `packaging/debian/` |
| `pkg_upstream_version` | `8.6.0~beta1` | Debian-ordered upstream version (`~` sorts before nothing) |
| `pkg_revision` | `1` | Debian package revision suffix |
| `publish` | `true` | Create/update a GitHub Release if build + smoke-test pass |

The distro is hardcoded (`jammy` / Ubuntu 22.04 / amd64).

## `bootstrap-packaging.yml` -- run once

This workflow is run **once** to set up `packaging/debian/` in this repo.
It clones the Debian php-team packaging for the last released PHP series
(e.g. 8.5), mechanically rewrites it for 8.6, and pushes to a branch
for review. After merging, `build-php86.yml` uses it for every build.

See the workflow file header for full details.

## Testing your own products before publishing

Add a `tests/smoke.sh` to this repository. It runs automatically in the
`smoke-test` job, after packages are installed and before anything is
published:

```bash
#!/usr/bin/env bash
set -euo pipefail
php8.6 -r "require 'vendor/autoload.php';"
composer --working-dir=/path/to/your/app check-platform-reqs
```

Anything that exits non-zero blocks publishing.

## What this does NOT do

- **No APT repository.** Packages are published as GitHub Release assets.
  Download and install with `dpkg -i`.
- **No isolated build chroot.** `dpkg-buildpackage` runs directly on the
  GitHub-hosted runner (still a fresh, disposable VM per run).
- **No guarantee of a clean first build.** The mechanical packaging
  rewrite from `bootstrap-packaging.yml` is best-effort.
- **Single architecture/distro.** `amd64` / jammy only, hardcoded.

## Safety note

These are pre-release, third-party-built PHP packages. `smoke-test`
(and your own `tests/smoke.sh`) is the only safety net before a build
is published. Be deliberate about which servers you install these on.
