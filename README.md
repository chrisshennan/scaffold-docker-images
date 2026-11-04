# Scaffold base images

Shared images for [Scaffold](https://buildwithscaffold.com) projects, so each project pulls them instead of compiling extensions and
Caddy on every build. They are rebuilt weekly to pick up upstream security patches.

## Images

There are two Docker Hub repositories, one per service, each with a tag per variant.

| Image | Tags | Contents |
|---|---|---|
| `chrisshennan/scaffold-php:<version>` | `8.4`, `8.5`, `8.4.27`, `8.5.12` | `php:X-fpm` + `intl`, `pdo_mysql`, OPcache, apt libs (acl, procps), production `php.ini` with `expose_php` off |
| `chrisshennan/scaffold-php:<version>-dev` | `8.4-dev`, `8.5-dev`, `8.4.27-dev`, `8.5.12-dev` | The above + git, unzip, xdebug, pcov, Composer, development `php.ini`, `expose_php` off. **Never use in production.** |
| `chrisshennan/scaffold-caddy:<version>` | `2.11.7` | Caddy + Brotli (cbrotli). For projects with fixed hostnames. |
| `chrisshennan/scaffold-caddy:<version>-route53` | `2.11.7-route53` | The above + the Route 53 DNS plugin, for wildcard certificates via DNS validation |

Tags:

- `8.5` follows the latest 8.5.x release (it moves to 8.5.13 and so on when that ships). `8.5-dev` does the same.
- `8.5.12` stays on that PHP patch version, but is still rebuilt weekly for OS and library patches.
- Caddy tags are exact versions, rebuilt weekly for OS and library patches.
- No tag is immutable, because every tag is rebuilt weekly. To pin an exact build, use its digest
  (`chrisshennan/scaffold-php@sha256:...`).

Pin by digest in production, since a base image change reaches every project that follows a tag.

Notes on the contents:

- **PHP:** the extensions are built against mysqlnd, so there is no MariaDB client library. The ini
  files are the ones PHP ships (`php.ini-production` / `php.ini-development`) plus
  `conf.d/scaffold-base.ini` (`expose_php = Off`). Projects add their own settings in `conf.d`, which loads
  after `scaffold-base.ini`.
- **pcov and xdebug** are both installed in the `-dev` image and both enabled, for code coverage in CI. Set
  `XDEBUG_MODE` (e.g. `coverage`, `off`) to control xdebug.
- **Caddy** is built with `xcaddy` and flattened to a single layer. The official image's runtime config
  (`ENV`, `EXPOSE`, `WORKDIR`, `CMD`) is re-declared in `caddy/Dockerfile`, and its OCI labels are not
  carried over. Both variants run as root, as the official image does.

## Using them in a project

Replace the project Dockerfile's `base`, `app-dev` and Caddy stages with `FROM` lines. Everything
project-specific (`composer install`, `COPY`, asset build, config) stays in the project.

A single parameterised base lets one set of project steps build both the production and the dev image:

```dockerfile
ARG PHP_IMAGE=chrisshennan/scaffold-php:8.5
FROM ${PHP_IMAGE} AS app
# ... composer install, COPY . /app, cache:clear, asset build ...
# dev / CI: docker build --target app --build-arg PHP_IMAGE=chrisshennan/scaffold-php:8.5-dev .

FROM chrisshennan/scaffold-caddy:2.11.7-route53 AS app-caddy   # or :2.11.7 without wildcard certs
COPY ./config/deploy/caddy /etc/caddy
COPY --from=app /app/public/ /app/public/
```

For local development the project can skip building the PHP image altogether: set the `php`
service in `compose.yaml` to `image: chrisshennan/scaffold-php:8.5-dev` (code is bind-mounted).

## Versions

Defined as build args with defaults:

- `PHP_VERSION` (php/Dockerfile): CI builds 8.4 and 8.5, the versions in PHP's active support. Revisit the
  matrix as versions leave active support (8.4's ends at the end of 2026; check php.net/supported-versions).
  The `8.5` / `8.5-dev` tags float, so weekly rebuilds pick up new minor/patch releases; CI fails if it resolves below
  `PHP_8_5_MIN_VERSION` (8.5.12) in the workflow. Raise that as patch releases ship.
- `CADDY_VERSION` (caddy/Dockerfile, and `CADDY_VERSION` in the workflow): 2.11.7. This is deliberately not
  the very latest release; the official Docker image for a new Caddy release can lag, so check that
  `caddy:<version>-alpine` and `caddy:<version>-builder-alpine` exist on Docker Hub before bumping.
- `CBROTLI_VERSION` (v1.0.1, the latest tag of `dunglas/caddy-cbrotli`) and `ROUTE53_VERSION` (v1.6.2).
  Check them against the latest releases and Caddy compatibility when bumping Caddy.
- Composer is pinned to a minor version (`composer:2.11` in `php/Dockerfile`); bump it by hand.

## Support policy

- **Rebuilt weekly:** the newest patch of each PHP release line in the build matrix (currently 8.4 and 8.5),
  and the Caddy version set in the workflow. Each rebuild picks up OS and library security patches.
- **Older patch tags are frozen:** once 8.5.13 ships, `8.5.12` stays on Docker Hub as it was last built but is
  no longer rebuilt or patched. Follow the minor tag (`8.5`, `8.5-dev`) to keep receiving patches; pin an
  exact patch tag or a digest only if you accept updating it yourself. The same applies to older Caddy tags.
- **Only actively supported PHP lines are built:** security-only lines (such as 8.3 at the time of writing)
  are not included. When a line leaves the matrix its tags freeze in the same way, so move to a newer line.

## Building locally

```bash
docker build --target php-dev --build-arg PHP_VERSION=8.5 -t php-dev:8.5 php
docker build --target caddy-route53 -t caddy-route53:2.11.7 caddy
```

Dockerfile targets: `php-base`, `php-dev`, `caddy`, `caddy-route53` (published as the plain, `-dev` and `-route53` tags). The builds need network access to Docker Hub,
`deb.debian.org`, `pecl.php.net`, `dl-cdn.alpinelinux.org`, `proxy.golang.org`, `sum.golang.org`,
`storage.googleapis.com` and `github.com`.

## Pipeline

`.github/workflows/build.yml` (GitHub Actions) builds a matrix of images, smoke-tests each on amd64
(required modules present; the plain `caddy` image must *not* contain `dns.providers.route53`), scans
with Trivy (fails on fixable CRITICAL vulnerabilities), then pushes multi-arch (amd64, arm64) images with
provenance and SBOM attestations to Docker Hub. It pushes from `main`, the weekly schedule (Mondays
03:00 UTC) and manual runs. Pull requests build and test without logging in to Docker Hub. Layer caching
is off on purpose, so the weekly rebuild always picks up fresh packages.

The actions are pinned to commit SHAs (with the version in a comment); Dependabot is configured to keep
them updated.

It pushes to `<namespace>/scaffold-php` and `<namespace>/scaffold-caddy`, using the `DOCKERHUB_NAMESPACE`
repository variable (`chrisshennan`) and the `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets (the token should be
an access token scoped to those two repositories only). The weekly schedule only
runs from the default branch.

## Security notes

- Only the `-route53` Caddy consumers should get AWS credentials. The plain image carries no DNS plugin.
- Scope the IAM policy to the one hosted zone (`ChangeResourceRecordSets`, `ListHostedZones`, `GetChange`).
- Treat registry push access as a security boundary: a compromised base image affects every project.
  Use a scoped Docker Hub access token, never a password, and never expose secrets to fork pull requests.
- The `-dev` PHP image contains debugging and coverage tooling and development ini settings. Do not deploy it.
- Pin the plugin versions, and pin the images by digest in production.

## License

MIT. See [LICENSE](LICENSE). Part of [Scaffold](https://buildwithscaffold.com). The images also contain third-party software (PHP, Caddy, Debian and Alpine
packages, xdebug, pcov, Composer) under its own licenses.
