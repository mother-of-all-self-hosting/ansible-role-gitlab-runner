<!--
SPDX-FileCopyrightText: 2018-2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer, although [20.04 may be problematic](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/ansible.md#supported-ansible-versions) if you run the Ansible playbook on it)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default, but some don't (e.g. Ubuntu 18.04) and require manual installation (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

Currently these testing scenarios are available:

### `default`

Installs GitLab Runner with two runners: one with the role's defaults, and one with most of the per-runner settings in use. Its name contains characters (`"` and `\`) which break `config.toml` unless the role escapes them.

There is no GitLab instance in this scenario, so the runners cannot pick up any jobs.

### What is verified

The systemd service is `Restart=always`, so it reports `active` even while the container crash-loops. The verification therefore:

- waits for GitLab Runner's metrics endpoint, which it only serves once it has loaded `config.toml`
- checks that GitLab Runner reports the version pinned by `gitlab_runner_version`, and the `concurrent` setting of the role's configuration
- parses `config.toml` and checks that it holds the configured runners and settings, and that only the runner's user may read it
- checks that the container runs as `gitlab_runner_uid:gitlab_runner_gid` with the Docker socket's group, without capabilities, with a read-only filesystem, and with SIGQUIT as its stop signal
- checks that the runner's user can use the Docker socket
- changes `config.toml` and checks that GitLab Runner reloads it without restarting, which the role relies on
- watches the service for 45 seconds to make sure it is not restarting

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```

If Docker Hub rate-limits you, set `MOLECULE_DOCKER_REGISTRY_MIRROR` to have the Docker daemon inside the test container use a registry mirror:

```bash
MOLECULE_DOCKER_REGISTRY_MIRROR=https://mirror.gcr.io molecule test --scenario-name default
```
