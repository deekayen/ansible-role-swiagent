# deekayen.swiagent

[![CI](https://github.com/deekayen/ansible-role-swiagent/actions/workflows/ci.yml/badge.svg)](https://github.com/deekayen/ansible-role-swiagent/actions/workflows/ci.yml) [![Ansible Galaxy](https://img.shields.io/badge/galaxy-deekayen.swiagent-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/deekayen/swiagent/) [![Project Status: Unsupported – The project has reached a stable, usable state but the author(s) have ceased all work on it. A new maintainer may be desired.](https://www.repostatus.org/badges/latest/unsupported.svg)](https://www.repostatus.org/#unsupported) ![BSD 3-Clause license](https://img.shields.io/badge/license-BSD%203--Clause-blue)

An Ansible role that installs the SolarWinds Orion agent (`swiagent`) on Enterprise Linux from the package repository your Orion server publishes, then initializes the agent against a poller and starts the `swiagentd` service.

The role writes a yum mirror list to `/etc/yum.repos.d/swiagent-rhel-5.mirrors` with one entry per `orion_hosts` URL, pointing at `Orion/AgentManagement/LinuxPackageRepository.ashx?path=/dists/rhel-5/$basearch`, and adds a `swiagent` repository that reads it. It installs `perl` for monitoring extensions, renders the agent settings to `/tmp/SolarWindsAgent.ini`, and installs the `swiagent` package. When the package install reports a change, handlers run `service swiagentd init` with that ini file, set `swiagent:swiagent` ownership on `swiagent_install_path`, and start and enable `swiagentd`.

## Requirements

- ansible-core 2.15 or newer on the controller.
- Privilege escalation on the target. Run the play with `become: true`; the role writes to `/etc/yum.repos.d` and installs packages.
- Fact gathering left on. The repository tasks branch on `ansible_facts.os_family`.
- Network access from the target to every URL in `orion_hosts`, and from the agent to the poller at `swiagent_target` on `swiagent_port`.

## Supported platforms

| Platform | Versions |
| --- | --- |
| EL | 9, 10 |

CI runs `ansible-lint` and `ansible-playbook --syntax-check` only.

## Installation

The role is listed on Galaxy as `deekayen.swiagent` but has no imported release yet, so install it from git with a `requirements.yml`:

```yaml
---
roles:
  - name: deekayen.swiagent
    src: https://github.com/deekayen/ansible-role-swiagent.git
    scm: git
    version: main
```

```bash
ansible-galaxy role install -r requirements.yml
```

## Role variables

Every `swiagent_*` variable except `swiagent_install_path` is written to the `[parameters]` section of `SolarWindsAgent.ini` under the key named in the description. The values are strings, including `"true"` and `"false"`.

| Variable | Default | Description |
| --- | --- | --- |
| `orion_hosts` | `[]` | Orion base URLs used as mirrors for the agent repository. Required; the role fails unless the list is non-empty and every entry starts with `http://` or `https://`. |
| `swiagent_installcert` | `"false"` | `installcert`. `"true"` or `"false"`. |
| `swiagent_run_provision` | `"true"` | `run_provision`. `"true"` or `"false"`. |
| `swiagent_relative_path` | `"ini"` | `relative_path`. |
| `swiagent_target` | `""` | `target`, the poller host name the agent reports to. Required. |
| `swiagent_port` | `"17778"` | `port`, the poller port. Must be between 1 and 65535. |
| `swiagent_ipaddress` | `""` | `ipaddress`, the poller IP address. Required. |
| `swiagent_target_device_id` | `00000000-0000-0000-0000-000000000000` | `targetDeviceID`. Falls back to the older `swiagent_targetDeviceID` variable when that is set, so existing inventories keep working. |
| `swiagent_is_active` | `"true"` | `is_active`. `"true"` or `"false"`. |
| `swiagent_server_http_port` | `"17790"` | `server_http_port`. Must be between 1 and 65535. |
| `swiagent_proxy_access_type` | `"disabled"` | `proxy_access_type`. |
| `swiagent_install_path` | `/opt/SolarWinds/Agent` | Agent install directory. The ownership handler sets `swiagent:swiagent` on it recursively. |

`vars/main.yml` holds the repository path on the Orion server and the ini file location, `/tmp/SolarWindsAgent.ini`. Neither is meant to be overridden.

## Behavior

- The `swiagent` repository is added with `gpgcheck: false`, so yum installs the agent package without a signature check.
- The handlers run only when the `swiagent` package install reports a change. Changing an ini variable on a host that already has the agent rewrites `/tmp/SolarWindsAgent.ini` but does not run `swiagentd init` again.
- `/tmp/SolarWindsAgent.ini` stays on the host after the run, mode `0644`.
- The repository tasks run only when `ansible_facts.os_family` is `RedHat`. The `perl` and `swiagent` package tasks run on every host and retry twice, two seconds apart.

## Dependencies

None.

## Example playbook

```yaml
---
- name: Install the SolarWinds Orion agent.
  hosts: monitored_linux
  become: true

  vars:
    orion_hosts:
      - https://orion.example.internal
      - https://orion-backup.example.internal
    swiagent_target: poller.example.internal
    swiagent_ipaddress: 192.0.2.10

  roles:
    - deekayen.swiagent
```

The host names are placeholders, and `192.0.2.10` is from the documentation address range. Set `swiagent_target` and `swiagent_ipaddress` to the same poller.

## Known issues

- `vars/main.yml:3` requests `/dists/rhel-5/$basearch` from Orion, and `tasks/main.yml:18` names the mirror file `swiagent-rhel-5.mirrors`, while `meta/main.yml` declares EL 9 and 10. Whether Orion serves an EL 9 or EL 10 agent build at that path is not verified.
- `handlers/main.yml:8` always passes `-is_active` to `service swiagentd init`, whatever `swiagent_is_active` is set to.

## Development

CI runs on every push to `main` and every pull request (see `.github/workflows/ci.yml`). It runs `ansible-lint --profile production` and syntax-checks `tests/test.yml`. To run the same checks locally:

```bash
pip3 install ansible-lint
ansible-lint --profile production
mkdir -p .ansible/roles && ln -sfn "$PWD" .ansible/roles/deekayen.swiagent
ANSIBLE_ROLES_PATH=.ansible/roles:~/.ansible/roles ansible-playbook --syntax-check tests/test.yml -i tests/inventory
```

The repository also has a `.pre-commit-config.yaml`; run `pre-commit run --all-files` before pushing.

### Repository layout

| Path | Purpose |
| --- | --- |
| `tasks/main.yml` | Input validation, repository setup, package installs, and ini file. |
| `tasks/assert.yml` | Checks `orion_hosts`, the poller settings, and the ports. |
| `handlers/main.yml` | Runs `swiagentd init`, fixes ownership, and starts `swiagentd`. |
| `templates/mirrorlist.j2` | Mirror list built from `orion_hosts`. |
| `templates/SolarWindsAgent.ini.j2` | Agent settings read by `swiagentd init`. |
| `defaults/main.yml` | Every user-facing variable. |
| `vars/main.yml` | Orion repository path and ini file location. |
| `tests/` | Syntax-check playbook and inventory used by CI. |

## Releases

Pushing a git tag runs `.github/workflows/release.yml`, which imports the tagged commit into Ansible Galaxy as `deekayen.swiagent`. The import needs a `GALAXY_API_KEY` repository or organization secret.

## License

BSD 3-Clause. See [LICENSE](LICENSE).

## Author

[David Norman](https://github.com/deekayen). Sponsorship links are in [.github/FUNDING.yml](.github/FUNDING.yml).
