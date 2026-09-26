<!--
SPDX-FileCopyrightText: 2023 Slavi Pantaleev
SPDX-FileCopyrightText: 2025 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# GitLab Runner Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [GitLab Runner](https://docs.gitlab.com/runner/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

It uses GitLab's official [container image](https://docs.gitlab.com/runner/install/docker/) (`gitlab/gitlab-runner`), and supports the [Docker executor](https://docs.gitlab.com/runner/executors/docker/) only. Jobs run in containers next to the runner's own, on the host's Docker daemon (via its socket).

For other executors, operating systems or autoscaling, see [riemers/ansible-gitlab-runner](https://github.com/riemers/ansible-gitlab-runner), which installs GitLab Runner from its packages.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [defaults/main.yml](defaults/main.yml) for the full list of supported options.

💡 See this [document](docs/configuring-gitlab-runner.md) for details about setting up the service with this role.

## Development

### pre-commit

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```

### Molecule

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

Refer to [this page](./molecule/README.md) for details about how to utilize it.
