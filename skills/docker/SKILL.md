---
name: docker
description: Build, run, debug, and maintain Docker-based projects with explicit commands, consistent resource names, secure images, and persistent databases. Use when a project uses Docker, when adding containerization, or when a local database needs a Docker fallback.
---

# Docker

Run projects that use Docker in Docker. Prefer a suitable existing local development database; otherwise run the database in Docker. Name every project resource you create.

## Discover the setup

- Inspect repository instructions, Dockerfiles, Compose files/overrides, `.dockerignore`, environment examples, package scripts, lockfiles, and CI. Establish service roles, dev/build targets, ports, database engine/version, and persistence paths.
- Check `docker context show` and `docker info`. Confirm the intended local development engine before creating resources; do not incidentally operate on a remote deployment or switch contexts.
- If Docker is used in the project, build/start the application and run relevant checks in containers. If only dependencies are containerized, add the smallest suitable application development target/configuration. Diagnose Docker failures or report the blocker instead of silently running the app on the host.
- If the project has no Docker setup and containerization was not requested, preserve its app workflow. A Docker database fallback alone does not require containerizing the app.
- Preserve working configuration and existing resources. Apply naming rules to new resources; migrate existing names only when needed for the task, preserving data and references.

## Explicit commands

Use `docker <object> <action> [options] [arguments]` in terminal commands, scripts, documentation, and CI. Prefer long option names.

| Object | Commands |
| --- | --- |
| Image | `docker image build`, `docker image pull`, `docker image tag`, `docker image inspect`, `docker image ls`, `docker image push`, `docker image rm` |
| Container | `docker container run`, `docker container create`, `docker container start`, `docker container stop`, `docker container exec`, `docker container logs`, `docker container inspect`, `docker container ls`, `docker container rm` |
| Volume | `docker volume create`, `docker volume inspect`, `docker volume ls`, `docker volume rm` |
| Network | `docker network create`, `docker network connect`, `docker network inspect`, `docker network ls`, `docker network rm` |

Do not use top-level shorthand aliases for object operations. Plugins retain their actual syntax: `docker compose --project-name atlas config`, `docker buildx build --tag atlas-api-image:dev --load .`. Do not invent object subcommands for plugins. Commands without object equivalents, such as `docker info` and `docker version`, keep their documented syntax. Use `docker compose`, never the legacy hyphenated executable.

## Name every resource

Use `<project>-<app>-<object>`:

- `project`: stable lowercase repository/Compose identifier, containing letters, digits, and hyphens, starting with a letter or digit. Include an environment/worktree identifier here when parallel environments need isolation.
- `app`: `client`, `api`, `db`, or `others`. Qualify roles when multiple services or resources would collide: `others-worker`, `db-postgres`, `api-cache`.
- `object`: `image`, `container`, `volume`, or `network`. Keep this suffix last, before an image tag.

Examples: `atlas-client-image:dev`, `atlas-api-container`, `atlas-db-volume`, `atlas-others-network`, `atlas-others-worker-container`. Shared networks use `others`.

- Explicitly tag every project-built image with `--tag` or Compose `image:`. Use immutable release/commit tags for releases and `:dev` for local iteration. Never leave project builds untagged or rely on implicit `latest`.
- Preserve upstream base/service references and versions/digests. For a third-party image used directly as a service, pull the upstream reference, add a project alias with `docker image tag`, and run that alias. Record its upstream version/digest and refresh deliberately. Base layers and Docker-managed caches do not require project aliases.
- Set `--name` on every created container, including temporary tests and migrations. Use explicit named volumes for persistence and dependency caches. Inspect image-declared `VOLUME` paths and mount them explicitly to avoid implicit anonymous volumes.
- Explicitly create/name user-defined networks. Docker-managed built-in networks, layers, and internal build artifacts are outside this naming convention.
- In Compose, set top-level `name:` or always supply `--project-name`; set service `image:` and `container_name:`; set top-level volume/network `name:` values. Attach services to declared networks or explicitly name the `default` network. Compose keys alone do not guarantee engine resource names.
- Fixed `container_name` prevents Compose scaling. Use distinct named services for replicas, or agree on an orchestration-compatible convention when scaling is requested.
- Inspect matching names before reuse. Do not replace unrelated resources or delete data to resolve collisions.

## Choose and connect the database

1. Identify the existing engine, compatible version, database name, and configuration without exposing credentials. Do not introduce another engine for convenience.
2. Prefer a healthy local development database authorized for this project and reachable from the app container. Verify connectivity from that container; host connectivity alone is insufficient. Never use production data sources for development, tests, seeds, or incidental migrations.
3. Container `localhost` refers to that container. Use `host.docker.internal` for a host database on Docker Desktop. On compatible Linux engines, add `--add-host host.docker.internal:host-gateway` or Compose `extra_hosts: ["host.docker.internal:host-gateway"]`. This does not make a database bound only to host loopback reachable. Check listener, authentication, and firewall configuration without broadly exposing the database.
4. If no suitable local database is reachable, run the same engine in Docker with a compatible pinned version, named persistent volume, shared named network, runtime credentials, and engine-native readiness check. Verify the selected image version's documented data directory, especially across major upgrades.
5. Between containers, use Compose service DNS names (such as `db`) or a direct-run network alias/container name and the container port. Publish a database port only when host tools need it; bind it to `127.0.0.1` by default.
6. Configure connection URLs for the chosen topology without committing secrets. Keep browser-facing API URLs separate from internal service URLs; browsers cannot resolve Compose service names.
7. Run necessary migrations through existing tooling inside a named container or with `docker container exec`. Initialization scripts usually do not rerun against existing data volumes. Isolate test data from development data.

## Build practices

- Select trusted, minimal bases compatible with the runtime and native dependencies. Do not choose Alpine/distroless solely for size when compatibility requires another base.
- Pin base/service versions and prefer verified digests for reproducible production builds. Update pins deliberately for security fixes. Preserve the package manager, lockfile, and frozen/locked installation behavior.
- Use multi-stage builds to exclude build tooling and development dependencies from runtime images. Keep development and production targets distinct and copy only required runtime artifacts into the final stage.
- Set `WORKDIR`. Copy dependency manifests/lockfiles before frequently changing source. Use BuildKit caches when useful; do not routinely disable caching. Combine package-index refresh/install in one layer and remove package-manager caches there.
- Keep contexts small with `.dockerignore`: exclude VCS data, host dependencies, generated output, local databases, credentials, and real environment files. Retain required build inputs and non-secret examples.
- Never bake secrets into `ARG`, `ENV`, copied files, or layers. Use BuildKit secret/SSH mounts for private build access and runtime secret files or the project's secret mechanism for application credentials. Avoid printing resolved secrets.
- Run as a non-root user with correctly owned writable directories. Prefer `COPY` with explicit ownership. Use exec-form `ENTRYPOINT`/`CMD` for signal delivery and an init process when child reaping requires it.
- Match deployment architecture and native dependencies. Use Buildx for multi-platform output when needed; verify platform support before forcing emulation.

## Runtime practices

- Keep one service responsibility per container. Bind app listeners to `0.0.0.0` inside containers. Publish only needed ports, bound to `127.0.0.1` for local development unless remote access is intended. `EXPOSE` does not publish ports.
- Use user-defined networks and DNS rather than hardcoded IPs, host networking, or legacy links. Separate backend-only traffic when appropriate.
- Persist databases/uploads in named volumes. Use bind mounts for development source without obscuring installed dependencies. Handle UID/GID ownership without `chmod 777`. Run production code from the image.
- Use readiness checks with tools present in the image. Compose startup dependencies should use `condition: service_healthy` where needed; apps still require bounded retry/backoff for later disconnects. Do not substitute fixed sleeps for readiness.
- Avoid privileged mode, Docker socket mounts, unnecessary capabilities, and broad host mounts. Apply `no-new-privileges`, capability drops, and read-only root filesystems with explicit writable paths where compatible; verify database entrypoints and permissions still work.
- Set suitable CPU/memory limits and log rotation for long-running services. Send logs to stdout/stderr. Choose restart policies deliberately; do not restart one-off migration jobs or hide failures in restart loops.
- Support graceful termination with a suitable stop timeout. An unhealthy status alone does not cause Docker to restart a container.
- Use current Compose specification syntax without obsolete top-level `version:`. Use profiles/overrides for optional tools and Docker databases so a host database selection does not also start a duplicate database.
- Do not push images, deploy, or mutate shared environments unless authorized by the task.

## Verification and cleanup

- Validate Compose with `docker compose --project-name <project> config --quiet`. Build the changed target, start required services, and verify readiness, logs, and real HTTP/database connectivity without printing secrets.
- Run relevant checks inside the app container with `docker container exec <project>-api-container <command>` or a named temporary container. Compose one-off runs must include `--name <project>-api-check-container` and `--rm`; give migration jobs distinct explicit names too.
- When changing persistence, verify survival across container recreation with disposable fixtures in an isolated database. Do not delete the user's volume to test persistence.
- Clean up only temporary resources created for verification. Preserve development database volumes. Avoid broad prune operations and volume-deleting teardown as routine cleanup; destructive data removal requires explicit authorization.
- Report services started, selected database topology, URLs, resource names, and verification results. State blockers and unverified behavior. Supply reproducible full-form start/stop commands without secrets.

## Official references

Check current documentation for version-sensitive features and image-specific data layouts:

- [Docker CLI object commands](https://docs.docker.com/reference/cli/docker/)
- [Docker build practices](https://docs.docker.com/build/building/best-practices/)
- [Compose services and runtime settings](https://docs.docker.com/reference/compose-file/services/)
- [Compose networking and host connectivity](https://docs.docker.com/compose/how-tos/networking/)
- [Compose volume names and external volumes](https://docs.docker.com/reference/compose-file/volumes/)
