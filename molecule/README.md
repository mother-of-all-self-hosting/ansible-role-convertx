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

Currently there is one testing scenario available.

### `default`

Tests a standard ConvertX installation, and then uses it.

ConvertX answers on HTTP long before it is usable, and answers with a redirect either way, so the scenario deliberately does not settle for "the unit is active and something responded". Started with no configuration at all, ConvertX still reports `/healthcheck` as ok and still redirects `/` with a 302 — to `/setup` rather than `/login` — and `Restart=always` reports the unit as active even while the container crash-loops.

The scenario therefore checks three things that can fail independently of each other:

- **the role's configuration reached the process** — the HS256 signature of the auth cookie ConvertX issues is recomputed against the `JWT_SECRET` the role rendered (with that variable unset, ConvertX signs with a `randomUUID()` it invents at boot), and `HTTP_ALLOWED` and `HIDE_HISTORY` are given non-default values whose effects are observed in the response headers and in the rendered page
- **the running image is the version `defaults/main.yml` pins** — read both from the line the process prints at startup and from the container's own `dist/package.json`
- **ConvertX actually converts** — a generated Markdown file carrying a marker is uploaded, converted to HTML through pandoc, downloaded, and required to contain both the marker and HTML structure that the input did not have

Negative controls surround those: the `/setup` redirect is asserted before an account exists and the `/login` redirect afterwards, unknown paths must return 404, `/register` must be refused, and the converted file must not be served to a request that carries no auth cookie.

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
