# ansible-role-passcli

[![CI](https://github.com/sgaunet/ansible-role-passcli/actions/workflows/ci.yml/badge.svg)](https://github.com/sgaunet/ansible-role-passcli/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Ansible role to install the [Proton Pass CLI](https://proton.me/pass) (`pass-cli`).

The binary is downloaded from Proton's official distribution site, verified
against the SHA256 checksum published in Proton's release manifest, and installed
to a single path on the managed host. The role is idempotent: it only downloads
and replaces the binary when the installed version differs from the requested
one.

## Requirements

- A Linux managed host on `x86_64`/`amd64` or `aarch64`/`arm64` (Proton publishes
  no other Linux builds).
- `become: true`, since the default install path is `/usr/local/bin`.
- Outbound HTTPS access to `proton.me` from the managed host.

No collections beyond `ansible.builtin` are needed. The role gathers the two
facts it needs (`ansible_system`, `ansible_architecture`) itself, so it also works
in plays that run with `gather_facts: false`.

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `passcli_version` | `latest` | Version to install: `latest`, or a pinned version such as `2.3.2` |
| `passcli_binary_name` | `pass-cli` | Name of the installed binary |
| `passcli_binary_path` | `/usr/local/bin/{{ passcli_binary_name }}` | Absolute install path |
| `passcli_binary_owner` | `root` | Owner of the installed binary |
| `passcli_binary_group` | `root` | Group of the installed binary |
| `passcli_binary_mode` | `0755` | Mode of the installed binary |
| `passcli_tmp_directory` | `/tmp` | Directory **on the managed host** used to stage the download |
| `passcli_manifest_url` | `https://proton.me/download/pass-cli/versions.json` | Proton release manifest |
| `passcli_download_base_url` | `https://proton.me/download/pass-cli` | Base URL downloads are built from |
| `passcli_download_retries` | `3` | Retries for the manifest fetch and the binary download |
| `passcli_download_delay` | `5` | Seconds between those retries |
| `passcli_checksum` | *(undefined)* | Optional explicit checksum, e.g. `sha256:abc…`. See below. |

### Checksum verification

Proton's manifest only publishes hashes for the **current** release, so:

- `passcli_version: latest` — the checksum is read from the manifest and enforced
  by `get_url`. Nothing extra to do.
- `passcli_version: "2.3.2"` (pinned) — there is no checksum to look up, so the
  download is not verified unless you supply one yourself via
  `passcli_checksum`, in the `<algorithm>:<hash>` form `get_url` expects:

  ```yaml
  - role: sgaunet.passcli
    vars:
      passcli_version: "2.3.2"
      passcli_checksum: "sha256:f95c6b39b45d96b670f249ccbb56b06b3a17d4579357d2d04c4ac64e4ffbeff7"
  ```

## Example Playbook

```yaml
# Install the latest version (checksum-verified)
- hosts: all
  become: true
  roles:
    - sgaunet.passcli

# Install a specific version, into a custom path
- hosts: all
  become: true
  roles:
    - role: sgaunet.passcli
      vars:
        passcli_version: "2.3.2"
        passcli_binary_path: "/opt/bin/pass-cli"
```

## Development

The toolchain is pinned with [mise](https://mise.jdx.dev) (Python, `uv`,
[Task](https://taskfile.dev)) plus `requirements.txt` for the Python side.
`mise install` creates `.venv` and populates it, so `ansible-lint`, `yamllint`
and `molecule` all resolve from one interpreter.

```bash
mise install          # installs the toolchain and populates .venv
task linter           # yamllint + ansible-lint
task test             # every molecule scenario
```

Molecule uses the Docker driver, so a working Docker daemon is required.

### Test scenarios

| Scenario | What it covers |
|---|---|
| `default` | Installs `latest` from a play with `gather_facts: false`, then asserts ownership/mode, that the reported version matches the manifest, and that the staging directory is cleaned up |
| `pinned` | Installs a deliberately older pinned version, proving the pinned code path is honoured |
| `upgrade` | Seeds an old version, then converges with `latest` and asserts the binary was replaced |

Every scenario runs Molecule's `idempotence` step, so a second converge must
report zero changes.

Run one scenario, or target another distribution:

```bash
task test-pinned
MOLECULE_DISTRO=rockylinux9 molecule test -s upgrade
```

`MOLECULE_DISTRO` selects the [geerlingguy](https://hub.docker.com/u/geerlingguy)
Ansible test image (default `rockylinux9`); CI runs the matrix against
`ubuntu2404` and `rockylinux9`.

## License

MIT

## Author Information

This role was created by [sgaunet](https://github.com/sgaunet).
