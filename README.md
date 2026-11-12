# era4s deployment

This repository contains the Docker Compose stack used by Portainer to run
[era4s](https://github.com/michael-baumanns87/era4s) on the NAS.

The application repository remains independent of Portainer. Its CI pipeline
builds and publishes the Docker image, then triggers this repository's
GitHub Actions workflow. The workflow calls the Portainer GitOps webhook.

## Portainer setup

1. In Portainer, create a new **Standalone** stack using **Git repository**.
2. Repository URL: `https://github.com/michael-baumanns87/era4s-deployment`
3. Reference: `refs/heads/main`
4. Compose path: `compose/docker-compose.yml`
5. Configure the stack environment variables listed below.
6. Enable **GitOps updates** and select **Webhook**.
7. Enable **Re-pull image** and **Force redeployment**.
8. Copy the generated GitOps webhook URL into the GitHub repository secret
   `PORTAINER_GITOPS_WEBHOOK` (Settings → Secrets and variables → Actions).

The webhook URL is a secret. Do not commit it to this repository or expose it
in CI logs.

## Stack environment

Configure these variables in Portainer when creating the stack:

| Variable | Example / default | Purpose |
| --- | --- | --- |
| `ERA4S_DATA_DIR` | `/volume1/docker/era4s-data` | Absolute persistent data path on the NAS; replace with a path that exists on your NAS |
| `APP_PORT` | `8080` | Host port |
| `APP_BIND_ADDRESS` | `0.0.0.0` | Host bind address |
| `APP_IMAGE` | `docker.io/michaelbaumanns/era4s:latest-amd64` | Published application image |
| `APP_VERSION` | `era4s:latest-amd64` | Version label passed to the application |
| `LOG_MAX_FILES` | `5d` | Application log retention |
| `LOG_MAX_SIZE` | `1m` | Application log rotation size |

Also configure the required era4s integration variables (SMAX and AI/Copilot
credentials) in Portainer's stack environment. Do not store credentials in
Git.

Redis is intentionally not included in this NAS stack by default; era4s uses its
file-cache fallback. This also avoids creating Redis volumes on NAS filesystems
that may have Docker overlay/xattr limitations.

The data directory must be an absolute path on the NAS because Docker resolves
bind-mount source paths on the host.

## Deployment flow

1. The `era4s` CI pipeline builds, tests and pushes
   `docker.io/michaelbaumanns/era4s:latest-amd64`.
2. After a successful push, it sends a GitHub `repository_dispatch` event to
   this repository (event type: `deploy`).
3. The workflow calls the Portainer GitOps webhook.
4. Portainer checks out this repository, re-pulls the configured image and
   redeploys the stack.

This keeps Portainer credentials and webhook details out of the application
repository. The application pipeline only needs a GitHub token with permission
to dispatch an event to this repository. Store it in the application CI as a
masked variable; never put it in source control.

## Local Compose

This repository is only for the Portainer-managed NAS stack. The local Compose
deployment in the `era4s` repository remains available and independent.
