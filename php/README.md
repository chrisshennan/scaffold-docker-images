# scaffold-php

PHP-FPM base images for [Scaffold](https://buildwithscaffold.com) projects, built on the official
[`php`](https://hub.docker.com/_/php) image with the extensions a Scaffold app needs already compiled, so your
project's build doesn't have to. Multi-arch (`linux/amd64`, `linux/arm64`) and rebuilt weekly for security patches.

Source, issues and the build workflow: https://github.com/chrisshennan/scaffold-docker-images

## Tags

| Tag | Image |
|---|---|
| `8.5`, `8.5.12` | Production base image |
| `8.5-dev`, `8.5.12-dev` | Development / CI image. **Never use in production.** |
| `8.4`, `8.4.27` | Production base image |
| `8.4-dev`, `8.4.27-dev` | Development / CI image. **Never use in production.** |

- `8.5` follows the latest 8.5.x release; `8.5.12` stays on that PHP patch version (the patch numbers shown here
  were current when this was written). Both are rebuilt weekly for OS and
  library security patches. The same applies to the `-dev` tags and to 8.4.
- No tag is immutable. To pin an exact build, use a digest: `chrisshennan/scaffold-php@sha256:...`.

## What's inside

**Production image** (`8.5`, `8.4`)
- `php:X-fpm` (Debian) with the `intl`, `pdo_mysql` and OPcache extensions (`pdo_mysql` uses mysqlnd)
- `acl` and `procps`
- PHP's production `php.ini`, with `expose_php` turned off (`conf.d/scaffold-base.ini`)

**Development image** (`8.5-dev`, `8.4-dev`): everything above, plus
- `git`, `unzip` and [Composer](https://getcomposer.org)
- `xdebug` and `pcov` (both enabled; set `XDEBUG_MODE` to control xdebug, for example `coverage` or `off`)
- PHP's development `php.ini`, with `expose_php` turned off

## Usage

```dockerfile
FROM chrisshennan/scaffold-php:8.5 AS app

COPY . /app
# ... composer install, asset build, cache warm-up ...
```

For local development and CI, point the same project steps at the dev image:

```dockerfile
ARG PHP_IMAGE=chrisshennan/scaffold-php:8.5
FROM ${PHP_IMAGE} AS app
# docker build --build-arg PHP_IMAGE=chrisshennan/scaffold-php:8.5-dev .
```

Or in `compose.yaml`, with the code bind-mounted:

```yaml
services:
  php:
    image: chrisshennan/scaffold-php:8.5-dev
    volumes:
      - .:/app
```

Add your own PHP settings as extra files in `/usr/local/etc/php/conf.d/`; they load after `scaffold-base.ini`.

## Support policy

- The newest patch of each PHP release line in the build matrix (currently 8.4 and 8.5) is rebuilt weekly.
- Older patch tags stay on Docker Hub but are frozen. Once 8.5.13 ships, `8.5.12` is no longer rebuilt, so follow
  the minor tag (`8.5`) to keep receiving patches.
- Only actively supported PHP lines are built. When a line leaves the matrix, its tags freeze in the same way.

## License

MIT. See the [LICENSE](https://github.com/chrisshennan/scaffold-docker-images/blob/main/LICENSE). The images also
contain third-party software (PHP, Debian packages, xdebug, pcov, Composer) under its own licenses.

Copyright (c) 2026 Chris Shennan (https://chrisshennan.com)
