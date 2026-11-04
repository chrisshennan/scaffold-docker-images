# scaffold-caddy

[Caddy](https://caddyserver.com) images for [Scaffold](https://buildwithscaffold.com) projects, built with Brotli
compression (`encode br`) and, in the `-route53` variant, the Route 53 DNS plugin for wildcard certificates.
Multi-arch (`linux/amd64`, `linux/arm64`) and rebuilt weekly for security patches.

Source, issues and the build workflow: https://github.com/chrisshennan/scaffold-docker-images

## Tags

| Tag | Image |
|---|---|
| `2.11.7` | Caddy + Brotli. For sites with fixed hostnames (normal HTTP certificate validation). |
| `2.11.7-route53` | The above + the [Route 53 DNS plugin](https://github.com/caddy-dns/route53), for wildcard certificates via DNS validation. |

- The tag is the exact Caddy version. Each tag is rebuilt weekly for OS and library security patches.
- No tag is immutable. To pin an exact build, use a digest: `chrisshennan/scaffold-caddy@sha256:...`.

## What's inside

Built from the official [`caddy`](https://hub.docker.com/_/caddy) Alpine image with `xcaddy`:
- [`dunglas/caddy-cbrotli`](https://github.com/dunglas/caddy-cbrotli) for Brotli compression (`encode gzip zstd br`)
- `-route53` only: [`caddy-dns/route53`](https://github.com/caddy-dns/route53). The plain image does not contain
  any DNS provider plugin.

The image keeps the official image's layout: the Caddyfile is read from `/etc/caddy/Caddyfile`, state lives in `/data`
and `/config`, and it exposes ports 80, 443 (TCP and UDP) and 2019. The image is flattened to a single layer. It runs
as root, as the official image does.

## Usage

```dockerfile
FROM chrisshennan/scaffold-caddy:2.11.7-route53
COPY ./Caddyfile /etc/caddy/Caddyfile
```

Mount a volume on `/data` so certificates survive container restarts:

```yaml
services:
  caddy:
    image: chrisshennan/scaffold-caddy:2.11.7
    ports: ["80:80", "443:443", "443:443/udp"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
volumes:
  caddy_data:
```

Example Caddyfile using Brotli, PHP-FPM, and (with the `-route53` image) a wildcard certificate:

```
example.com, *.example.com {
    encode gzip zstd br

    tls {
        dns route53 {
            profile default
        }
    }

    root * /app/public
    php_fastcgi php:9000
    file_server
}
```

For the `-route53` image, AWS credentials are read the way the plugin documents (see
https://github.com/caddy-dns/route53). Give them only to that container, and scope the IAM policy to the one hosted
zone (`ChangeResourceRecordSets`, `ListHostedZones`, `GetChange`).

## Support policy

- The Caddy version set in the build workflow is rebuilt weekly.
- Older version tags stay on Docker Hub but are frozen. When the version moves on (for example to 2.11.8 or later),
  the old tag is no longer rebuilt or patched, so move to the new tag.

## License

MIT. See the [LICENSE](https://github.com/chrisshennan/scaffold-docker-images/blob/main/LICENSE). The images also
contain third-party software (Caddy, Alpine packages, the plugins above) under its own licenses.

Copyright (c) 2026 Chris Shennan (https://chrisshennan.com)
