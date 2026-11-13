# era4s deployment

This repository contains the Docker Compose stack used by Portainer to run
[era4s](https://github.com/michael-baumanns87/era4s) on the NAS.

The application repository builds, tests and publishes an immutable image tag,
then triggers this repository's GitLab pipeline with that exact image reference.
This repository commits the new reference to the Compose file before calling
the Portainer GitOps webhook. The image is the single source of truth for the
deployed application version; no separate `APP_VERSION` variable is maintained.

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
as `ERA4S_DEPLOY_PROJECT_ID`. The application pipeline passes only
`APP_IMAGE` (the immutable image reference) as a trigger-pipeline variable.

The deployment pipeline commits the Compose update using its `CI_JOB_TOKEN`.
In the deployment project's GitLab settings, enable **Allow Git push requests
to the repository** for CI job tokens. This is separate from the pipeline
trigger token. The deployment pipeline only runs for `trigger` and `web`
pipeline sources, so its own Git push does not recursively trigger another
deployment.

Do not commit webhook URLs or tokens, and do not print them in CI logs.

## Stack environment

Configure these variables in Portainer when creating the stack:

| Variable | Example / default | Purpose |
| --- | --- | --- |
| `ERA4S_DATA_DIR` | `/volume1/docker/era4s-data` | Absolute persistent data path on the NAS; replace with a path that exists on your NAS |
| `APP_PORT` | `8080` | Host port |
| `APP_BIND_ADDRESS` | `0.0.0.0` | Host bind address |
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

1. The `era4s` pipeline builds and tests the image.
2. Its deploy job pushes the image and exports the exact immutable registry
   reference (including the commit-derived tag) as a dotenv artifact.
3. The trigger job waits for that successful deploy job and starts this
   project's pipeline with only `APP_IMAGE`.
4. This pipeline commits the exact image reference to
   `compose/docker-compose.yml`.
5. Only after the commit succeeds, it calls `PORTAINER_GITOPS_WEBHOOK`.
6. Portainer checks out the new repository commit, re-pulls the pinned image
   and redeploys the stack. The running image supplies the application's
   version metadata.

The local Compose deployment in the `era4s` repository remains independent.
