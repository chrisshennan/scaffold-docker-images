# Shared Caddy images for Scaffold projects. Two variants, both with Brotli compression:
#
#   caddy          - fixed hostnames (default HTTP certificate validation)
#   caddy-route53  - adds the Route 53 DNS plugin, for wildcard certificates via DNS validation
#
# Build one with e.g.: docker build --target caddy-route53 -t caddy-route53:2.11.7 .
ARG CADDY_VERSION=2.11.7
ARG CBROTLI_VERSION=v1.0.1
ARG ROUTE53_VERSION=v1.6.2

FROM caddy:${CADDY_VERSION}-builder-alpine AS builder-base

# C tool-chain + Brotli headers/libs
RUN apk add --no-cache build-base brotli-dev pkgconfig

# cgo is off by default in the builder image
ENV CGO_ENABLED=1

FROM builder-base AS builder-plain

ARG CADDY_VERSION
ARG CBROTLI_VERSION

RUN xcaddy build v${CADDY_VERSION} \
    --with github.com/dunglas/caddy-cbrotli@${CBROTLI_VERSION}

FROM builder-base AS builder-route53

ARG CADDY_VERSION
ARG CBROTLI_VERSION
ARG ROUTE53_VERSION

RUN xcaddy build v${CADDY_VERSION} \
    --with github.com/caddy-dns/route53@${ROUTE53_VERSION} \
    --with github.com/dunglas/caddy-cbrotli@${CBROTLI_VERSION}

FROM caddy:${CADDY_VERSION}-alpine AS runtime-base

# The upstream image already ships brotli-libs, so upgrade to pick up security fixes published since it was built
RUN apk upgrade --no-cache && apk add --no-cache brotli-libs

# Each variant is assembled, then flattened into a single layer. Copying the custom binary over the one in
# the official image would otherwise leave the original binary behind in a lower layer (~55 MB of dead weight).
# The official image's runtime config is re-declared by hand below, so check it when bumping CADDY_VERSION.
FROM runtime-base AS caddy-assembled

COPY --from=builder-plain /usr/bin/caddy /usr/bin/caddy

FROM runtime-base AS caddy-route53-assembled

COPY --from=builder-route53 /usr/bin/caddy /usr/bin/caddy

FROM scratch AS caddy-base

ARG CADDY_VERSION
ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
    CADDY_VERSION=v${CADDY_VERSION} \
    XDG_CONFIG_HOME=/config \
    XDG_DATA_HOME=/data
EXPOSE 80 443 443/udp 2019
WORKDIR /srv
CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]

FROM caddy-base AS caddy

COPY --from=caddy-assembled / /

FROM caddy-base AS caddy-route53

COPY --from=caddy-route53-assembled / /
