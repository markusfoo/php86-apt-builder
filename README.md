# PHP APT Builder

Pre-built `.deb` packages for PHP pre-release versions (beta/RC), published as
**GitHub Releases** with all assets attached. No APT repository setup
required -- just download the `.deb` files and install.

Built from the **real Debian php-team packaging**, adapted for the target
PHP series. Produces the same split of packages that `deb.sury.org` publishes
(`phpX.Y-cli`, `phpX.Y-fpm`, `phpX.Y-common`, `phpX.Y-gd`, `phpX.Y-mysql`,
`phpX.Y-curl`, `phpX.Y-mbstring`, `phpX.Y-xml`, `phpX.Y-zip`, etc.).

**PECL extensions** are built automatically as part of every release:
xdebug, redis, igbinary, msgpack, apcu, mongodb, imagick, memcached.
Extensions that fail to compile are skipped with a warning.

Target: **Ubuntu 22.04 (jammy) / amd64**.

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
| **build** | Compiles all PHP `.deb` packages |
| **smoke-test** | Installs in a clean container and runs `php -v`, `php -m`, `php-fpm -t` |
| **build-pecl** | Builds PECL extensions against the compiled PHP (parallel with smoke-test) |
| **publish** | Uploads new `.deb` files to the GitHub Release (smart upsert) |

### PECL extensions

Built automatically in the `build-pecl` job. To add or remove extensions,
edit the `EXTENSIONS` variable in `build-php.yml`.

Imagick and msgpack include build-time patches for PHP 8.6 API compatibility.

### Smart release management

The `publish` job does **not** delete and recreate releases. Instead:

- If a release for this version tag already exists, only **new** `.deb`
  files are uploaded (existing assets are kept).
- Beta/RC/alpha versions are automatically marked as pre-release.

## `build-php.yml` -- the day-to-day build

| Input | Default | Meaning |
|---|---|---|
| `php_tag` | `php-8.6.0beta1` | Git tag in `php/php-src` to build |
| `target_series` | `8.6` | PHP series -- must match `packaging/debian/` |
| `pkg_upstream_version` | `8.6.0~beta1` | Debian-ordered upstream version (`~` sorts before nothing) |
| `pkg_revision` | `1` | Debian package revision suffix |
| `publish` | `true` | Create/update a GitHub Release if build + smoke-test pass |

## `bootstrap-packaging.yml` -- run once

Run **once** to set up `packaging/debian/`. It clones the Debian php-team
packaging for the last released PHP series, rewrites it for the target series,
and pushes to a branch for review.

## `extension-checker.yml` -- fast compile test

Downloads pre-built PHP `.deb` packages from a release, installs them, and
tests compilation of each PECL extension. Completes in ~2 minutes. Use this
to iterate on patches before running the full build.

---

## Changelog

### 8.6.0 Beta 1

- Initial project setup: Debian packaging adopted from `debian/main/8.5`
- Fixed 11 Debian patches for PHP 8.6.0beta1 API changes
- Fixed session.save_path default to `/var/lib/php/sessions`
- Fixed lintian: removed dh-systemd (not on Ubuntu 22.04)
- Fixed CI: chmod +x on shtool/config-stubs/gen_stub.php, --enable-pic, Node 24 env
- Fixed CI: hidden files in upload-artifact, ppa:ondrej/php in smoke-test, PGDG repo
- Fixed CI: apt retries/timeouts, bypass flaky azure.archive.ubuntu.com mirror
- Switched from branch-based apt repo to GitHub Releases
- Integrated PECL extension building into main workflow (8 extensions)
- Fixed PECL: checkinstall --install=no, file permissions, extension filtering
- Added extension-checker workflow for fast compile testing
- Fixed extension-checker: msgpack `\0` sed escape, 3 extension build failures
- Rewrote PECL builder to clone from official GitHub repos instead of PECL
- Fixed dpkg-deb version error: extract version from `package.xml` instead of git tags
- Simplified release tag format: `php8.6.0-beta1-1` (universal for any PHP version)
- Renamed repo and workflows to be version-agnostic (`build-php.yml`, `php-apt-builder`)
