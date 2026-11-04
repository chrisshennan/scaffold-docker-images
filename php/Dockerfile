# Shared PHP images for Scaffold projects.
#
#   php-base - php-fpm plus the extensions every Scaffold project needs
#   php-dev  - php-base plus the local-development / CI tooling
#
# Build one with e.g.: docker build --target php-dev --build-arg PHP_VERSION=8.5 -t php-dev:8.5 .
ARG PHP_VERSION=8.5

FROM php:${PHP_VERSION}-fpm AS php-base

# apt-get upgrade picks up security fixes published since the upstream php image was last built.
# - libicu-dev: Needed only to compile the intl extension; purged afterwards (the runtime libicu stays)
# - acl: For setting filesystem permissions
# - procps: For process management utilities
# pdo and pdo_mysql are built against mysqlnd, so no MariaDB client library is needed.
RUN apt-get update \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends libicu-dev acl procps \
    && docker-php-ext-install -j"$(nproc)" pdo_mysql intl \
    # OPcache is compiled into PHP 8.5+ but must be installed on earlier versions.
    && (php --ri "Zend OPcache" > /dev/null 2>&1 || docker-php-ext-install opcache) \
    # Keep the shared libraries the compiled extensions link against, then drop the -dev packages.
    && find /usr/local -type f -executable -exec ldd '{}' ';' \
        | awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); print so }' \
        | sort -u \
        | xargs -r dpkg-query --search \
        | grep -v '^diversion by' \
        | cut -d: -f1 | sort -u \
        | xargs -r apt-mark manual \
    && apt-get purge -y --auto-remove libicu-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Production defaults (display_errors off, error logging on). Projects override via conf.d/*.ini.
RUN cp "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

# php.ini-production leaves expose_php on, which advertises the PHP version in response headers.
# In conf.d so it also applies to php-dev; it loads before a project's own conf.d files (alphabetical).
RUN echo "expose_php = Off" > "$PHP_INI_DIR/conf.d/scaffold-base.ini"

WORKDIR /app

FROM php-base AS php-dev

# Development defaults (display_errors on) replace the production ini inherited from php-base.
RUN cp "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"

RUN apt-get update && apt-get install -y --no-install-recommends \
        git \
        unzip \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN pecl install xdebug pcov \
    && docker-php-ext-enable xdebug pcov \
    && rm -rf /tmp/pear

# Tagged so this pulls the image rather than resolving to a build stage named "composer"
COPY --from=composer:2.11 /usr/bin/composer /usr/local/bin/composer
