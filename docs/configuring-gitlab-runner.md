<!--
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up GitLab Runner

This is an [Ansible](https://www.ansible.com/) role which installs [GitLab Runner](https://docs.gitlab.com/runner/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

GitLab Runner runs the CI/CD jobs of a GitLab instance. This role supports the [Docker executor](https://docs.gitlab.com/runner/executors/docker/) only: every job runs in a container of its own.

See the project's [documentation](https://docs.gitlab.com/runner/) to learn what GitLab Runner does and why it might be useful to you.

## How it works

The runner's container gets the host's Docker socket, and starts the containers of its jobs through it. They run next to the runner's container (not inside it), on the host's Docker daemon, just as with a GitLab Runner installed from a package. This is the setup GitLab [documents](https://docs.gitlab.com/runner/install/docker/) for running GitLab Runner in a container.

>[!WARNING]
> Access to the Docker socket is equivalent to `root` access on the host. The runner's container itself runs as an unprivileged user, without capabilities and with a read-only filesystem, but anyone who can make GitLab Runner start containers can take over the host, and so can the jobs of a runner with `docker_privileged: true`.
>
> Run the runner on a host of its own (not the one running GitLab or other services), and only let it run jobs you trust.

The role writes GitLab Runner's `config.toml` itself, from the runners you define. It does not run `gitlab-runner register`.

## Prerequisites

Each runner needs to be created in GitLab first:

- an instance runner in **Admin area** → **CI/CD** → **Runners** → **New instance runner**
- a group runner in the group's **Build** → **Runners** → **New group runner**
- a project runner in the project's **Settings** → **CI/CD** → **Runners** → **New project runner**

GitLab then shows the runner's authentication token, which starts with `glrt-`. Ignore the `gitlab-runner register` command it suggests.

Registration tokens (which start with `GR1348941`) are deprecated by GitLab and not supported by this role.

>[!NOTE]
> If your GitLab instance sets an expiration for runner authentication tokens, GitLab Runner does not rotate the tokens this role configures, as `config.toml` does not record when they expire. Create a new token for the runner in GitLab before the old one expires, and update `gitlab_runner_runners`.

## Adjusting the playbook configuration

To enable GitLab Runner, add the following configuration to your `vars.yml` file (e.g. `inventory/host_vars/mash.example.com/vars.yml` with the [MASH playbook](https://github.com/mother-of-all-self-hosting/mash-playbook)):

```yaml
########################################################################
#                                                                      #
# gitlab_runner                                                        #
#                                                                      #
########################################################################

gitlab_runner_enabled: true

gitlab_runner_config_gitlab_url: https://gitlab.example.com

gitlab_runner_runners:
  - name: docker
    token: YOUR_RUNNER_AUTHENTICATION_TOKEN_HERE

########################################################################
#                                                                      #
# /gitlab_runner                                                       #
#                                                                      #
########################################################################
```

Each item of `gitlab_runner_runners` is a runner, which can override `url` and a few common settings of the Docker executor (the image jobs run in by default, volumes, privileged mode, the network of job containers). See `gitlab_runner_runners` in [`defaults/main.yml`](../defaults/main.yml) for all of them.

The number of jobs that run at the same time, across all runners, is limited by `gitlab_runner_config_concurrent` (default: `4`).

GitLab Runner buffers the log of each running job in its container's `/tmp`, which holds 512 MB (`gitlab_runner_container_tmp_size`). That is enough for 128 jobs at the default `output_limit` of 4 MB. If `gitlab_runner_config_concurrent` times `output_limit` gets close to it, raise `gitlab_runner_container_tmp_size`.

### Adding other settings

GitLab Runner has [many more settings](https://docs.gitlab.com/runner/configuration/advanced-configuration/). To add them, use:

- `gitlab_runner_configuration_extension_toml` for the global section
- a runner's `docker_configuration_extension_toml` for its `[runners.docker]` section
- a runner's `configuration_extension_toml` for the runner itself (e.g. `output_limit`), and for its other sections (e.g. `[runners.cache]`). Plain keys must come before any section.

For example:

```yaml
gitlab_runner_runners:
  - name: docker
    token: YOUR_RUNNER_AUTHENTICATION_TOKEN_HERE
    docker_configuration_extension_toml: |
      pull_policy = ["always", "if-not-present"]
      allowed_images = ["alpine:*", "python:*"]
    configuration_extension_toml: |
      output_limit = 16384
      [runners.cache]
        Type = "s3"
        Shared = true
        [runners.cache.s3]
          ServerAddress = "s3.example.com"
          BucketName = "runner-cache"
```

GitLab Runner reloads `config.toml` on its own when it changes. So changing these settings does not restart it, and does not interrupt running jobs.

>[!NOTE]
> Paths in `config.toml` (e.g. bind mounts in `docker_volumes`) refer to the host, because the job containers are started by the host's Docker daemon.

### Building container images in jobs (Docker-in-Docker)

Jobs that build container images with a [`docker:dind` service](https://docs.gitlab.com/ci/docker/using_docker_build/#use-docker-in-docker) need a runner with privileged containers:

```yaml
gitlab_runner_runners:
  - name: docker-privileged
    token: YOUR_RUNNER_AUTHENTICATION_TOKEN_HERE
    docker_privileged: true
    docker_volumes: ["/cache", "/certs/client"]
```

A privileged job container has `root` access to the host. Where possible, build images without privileges instead, e.g. with [BuildKit in rootless mode](https://docs.gitlab.com/ci/docker/using_buildkit/) or [Buildah](https://docs.gitlab.com/ci/docker/buildah_rootless_tutorial/), on a runner without `docker_privileged`. Otherwise, give the privileged runner a tag (in GitLab), so that only the jobs which need it run on it.

### Using a GitLab instance on the same host (optional)

If GitLab (e.g. installed with [ansible-role-gitlab](https://github.com/spatterIight/ansible-role-gitlab)) runs on the same host, the runner can use its public URL as usual.

To reach it through its container network instead (e.g. because the public URL isn't reachable from the host itself), connect the runner and its job containers to that network, and point them at the container:

```yaml
gitlab_runner_container_additional_networks_custom:
  - gitlab

gitlab_runner_config_gitlab_url: http://gitlab

gitlab_runner_runners:
  - name: docker
    token: YOUR_RUNNER_AUTHENTICATION_TOKEN_HERE
    docker_network_mode: gitlab
```

Jobs clone repositories from the URL GitLab advertises (its `external_url`). If that isn't reachable from the job containers, set `clone_url` (or `gitlab_runner_config_gitlab_clone_url`) to `http://gitlab` too.

### Exposing metrics (optional)

To have GitLab Runner serve [Prometheus metrics](https://docs.gitlab.com/runner/monitoring/), and publish them on the host's loopback interface:

```yaml
gitlab_runner_config_metrics_enabled: true

gitlab_runner_container_metrics_host_bind_port: "127.0.0.1:9252"
```

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Stopping and upgrading

The container is stopped with SIGQUIT, on which GitLab Runner stops picking up new jobs and waits for the running ones to finish. After `gitlab_runner_container_stop_grace_time_seconds` (default: 10 minutes), the jobs still running are killed and fail.

The role restarts the container only when the container image (e.g. a new GitLab Runner version) or the systemd service changes, not when only `config.toml` changes.

GitLab recommends keeping the major and minor version of GitLab Runner in sync with that of GitLab. Other combinations may work, but some features may not. See [this page](https://docs.gitlab.com/runner/#gitlab-runner-versions) for details. To pin another version, set `gitlab_runner_version` (e.g. `gitlab_runner_version: 19.3.3`).

## Uninstalling

Setting `gitlab_runner_enabled: false` stops the service and removes its files (`gitlab_runner_base_path`). The runners stay in GitLab, and can be deleted there.

## Troubleshooting

### Check the service's logs

Run `journalctl -fu gitlab-runner` on the server (or the name of your service, e.g. `mash-gitlab-runner`).

### List and verify the configured runners

```sh
docker exec gitlab-runner gitlab-runner list --config /etc/gitlab-runner/config.toml
docker exec gitlab-runner gitlab-runner verify --config /etc/gitlab-runner/config.toml
```
