# TODO

## Before the first push
- [x] Choose a license, add a `LICENSE` file, and add the `org.opencontainers.image.licenses` label to the images (MIT).
- [x] Create the Docker Hub repositories `chrisshennan/scaffold-php` and `chrisshennan/scaffold-caddy`.
- [x] Create a Docker Hub access token (Read & Write, scoped to those two repositories if the plan allows).
- [x] Set the `DOCKERHUB_NAMESPACE` repository variable and the `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets.
- [x] Rebuild and re-check the PHP 8.4 images (`-base`, `-dev`) after `expose_php = Off` was added.
- [x] Check `git` over HTTPS works inside the `-dev` image (`git ls-remote` against github.com works; a full
      Composer source install was not run because packagist is blocked in this sandbox).
- [x] Check whether the Caddy strip flags (`XCADDY_GO_BUILD_FLAGS`) take effect. They made the binary larger (54.2 vs
      48.6 MB for the plain image), so they were removed; xcaddy's default already strips.

## First pipeline runs
- [ ] Run the GitHub Actions workflow on a pull request (build, smoke tests and Trivy scan; nothing is pushed).
- [ ] Check the multi-arch (amd64 + arm64) build and the first push to Docker Hub from `main`.
- [ ] Confirm the weekly schedule runs once the workflow is on `main` (schedules only run from the default branch).

## Using the images
- [ ] Try the parameterised `PHP_IMAGE` Dockerfile pattern (README, "Using them in a project") in noticeboards.info.
- [ ] Switch noticeboards.info's `Dockerfile` and `compose.yaml` over to the published images.

## Later
- [ ] Revisit the PHP matrix when 8.4 leaves active support (end of 2026; confirm at php.net/supported-versions).
- [ ] Move to Caddy 2.11.8 (or later) once its official Docker images are published, then re-check the
      re-declared runtime config in `caddy/Dockerfile`.
- [ ] Consider running Caddy as a non-root user (needs `cap_net_bind_service` and a check of volume ownership).
