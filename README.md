# era4s deployment

This repository contains the Docker Compose stack used by Portainer to run
[era4s](https://github.com/michael-baumanns87/era4s) on the NAS.

The application repository remains independent of Portainer. Its GitLab CI
pipeline builds, tests and publishes the Docker image, then triggers a pipeline
in this repository. This repository's GitLab pipeline calls the Portainer
GitOps webhook. Portainer-specific credentials stay here, not in `era4s`.

## GitLab and Portainer setup

1. Import this repository into the NAS GitLab instance as
   `era4s-deployment`. Use the NAS GitLab project URL as the canonical
   repository from then on.
2. In Portainer, create a new **Standalone** stack using **Git repository**.
3. Set the repository URL to the NAS GitLab clone URL of this project.
4. Reference: `refs/heads/main`.
5. Compose path: `compose/docker-compose.yml`.
6. Configure the stack environment variables listed below.
7. Enable **GitOps updates** and select **Webhook**.
8. Enable **Re-pull image** and **Force redeployment**.
9. Copy the generated GitOps webhook URL into this GitLab project's CI/CD
   variable `PORTAINER_GITOPS_WEBHOOK` (masked; protected if the deployment
   branch is protected).

Create a **pipeline trigger token** in this deployment project. Store that
token in the `era4s` GitLab project as the masked/protected CI/CD variable
`ERA4S_DEPLOY_TRIGGER_TOKEN`. Also store the numeric deployment project ID
as `ERA4S_DEPLOY_PROJECT_ID`. The application pipeline uses these values to
trigger this project's `main` pipeline through the GitLab API.

Do not commit webhook URLs or tokens, and do not print them in CI logs.

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

1. The `era4s` GitLab pipeline builds, tests and pushes
   `docker.io/michaelbaumanns/era4s:latest-amd64`.
2. After a successful push, it triggers this project's GitLab pipeline using
   `ERA4S_DEPLOY_PROJECT_ID` and `ERA4S_DEPLOY_TRIGGER_TOKEN`.
3. This project's pipeline calls `PORTAINER_GITOPS_WEBHOOK`.
4. Portainer checks out this repository, re-pulls the configured image and
   redeploys the stack.

The local Compose deployment in the `era4s` repository remains available and
independent.
