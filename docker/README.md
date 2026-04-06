# Docker

Containerization fundamentals, Node.js best practices, and production-ready patterns.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Dockerfile Best Practices](#dockerfile-best-practices)
- [Multi-Stage Builds](#multi-stage-builds)
- [Docker Compose V2](#docker-compose-v2)
- [Networking](#networking)
- [Volumes and Persistence](#volumes-and-persistence)
- [Health Checks](#health-checks)
- [Node.js in Docker](#nodejs-in-docker)
- [Security](#security)
- [Production Patterns](#production-patterns)

---

## Core Concepts

<!-- Cover: images vs containers, layers, registries, tags, build context -->

TODO

## Dockerfile Best Practices

<!-- Cover: ordering for layer caching, .dockerignore, reducing image size, pinning base image versions -->

TODO

## Multi-Stage Builds

<!-- Cover: builder pattern, separating build and runtime deps, copying artifacts between stages -->

TODO

## Docker Compose V2

<!-- Cover: compose.yaml syntax, services, depends_on with healthcheck, profiles, watch mode -->

TODO

## Networking

<!-- Cover: bridge vs host vs overlay, service discovery, exposing ports, internal networks -->

TODO

## Volumes and Persistence

<!-- Cover: named volumes, bind mounts, tmpfs, volume drivers, backup strategies -->

TODO

## Health Checks

<!-- Cover: HEALTHCHECK instruction, curl vs wget vs custom, intervals, compose integration -->

TODO

## Node.js in Docker

<!-- Cover: choosing base image (alpine vs slim vs bookworm), npm ci, non-root user, signal handling (tini), .dockerignore for node_modules -->

TODO

## Security

<!-- Cover: non-root users, read-only filesystem, secrets management, scanning with trivy/scout, distroless images -->

TODO

## Production Patterns

<!-- Cover: logging (stdout/stderr), graceful shutdown, resource limits, orchestration basics (Swarm/K8s mention) -->

TODO

---

## References

- [Docker Docs](https://docs.docker.com)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Node.js Docker Best Practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md)
