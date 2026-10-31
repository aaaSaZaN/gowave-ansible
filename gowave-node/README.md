# GoWave node with Ansible

This playbook installs pinned GoWave node and Xray releases on Linux x86_64 or
aarch64 hosts with systemd. It follows the structure of `../gowave`, verifies the
node binary and ASN index against `checksums.txt`, and verifies the Xray archive
against its GitHub release asset SHA-256 digest. Xray's executable and bundled
`geoip.dat`/`geosite.dat` are installed alongside the node. No Go or container
runtime is needed on the target.

Targets need Python 3, SSH access, privilege escalation to root, working package
repositories, and HTTPS access to GitHub releases and `api.github.com`. Package
installation covers Debian, RedHat, and Archlinux families, including nftables
and iproute tools. The role does not enable the distribution's nftables service
or replace its firewall rules.

## Deploy

Run these commands from this directory with [uv](https://docs.astral.sh/uv/)
installed on the controller:

```sh
uv sync --locked
cp inventory.example.yml inventory.local.yml
```

Edit `inventory.local.yml` to set the SSH host/user, `gowave_node_version`, and
`gowave_node_core_version`. Both versions must be existing release tags with the
`v` prefix, such as `vX.Y.Z`; prerelease tags are accepted. There is no `latest`
resolution. The node release must publish `checksums.txt`, `asn-prefixes.zip`,
and `gowave-node-linux-amd64` or `gowave-node-linux-arm64`. The Xray release must
publish the matching `Xray-linux-64.zip` or `Xray-linux-arm64-v8a.zip` with a
SHA-256 asset digest.

Create a node in the GoWave panel and copy its `SECRET_KEY` into an encrypted
variables file. This is the panel-generated certificate/JWT payload, not a newly
generated random password:

```sh
uv run --locked ansible-vault create secrets.local.yml
```

Enter this variable with the value from the panel:

```yaml
vault_gowave_node_secret_key: PASTE_PANEL_SECRET_KEY_HERE
```

The example inventory references that value through `gowave_node_secret_key`.
Local inventory and secret files ending in `.local.yml` or `.local.yaml` are
ignored by Git. Configuration tasks hide values from logs and diffs; the remote
`.env` is root-owned with mode `0600`.

```sh
uv run --locked ansible-playbook -i inventory.local.yml playbook.yml \
  -e @secrets.local.yml --ask-vault-pass --ask-become-pass
```

Omit `--ask-become-pass` for passwordless sudo. For multiple nodes, add hosts to
the `gowave_node` group and set each host's secret and port in host variables.

In the panel, use the node's reachable address and the same management port
(default `3000`). Allow that TCP port from the panel and allow the Xray inbound
ports required by your configuration. Management uses the node's native TLS 1.3,
mTLS, and JWT authentication. DNS and host/provider firewall access rules are
managed separately.

## Configuration

Defaults are in [roles/gowave_node/defaults/main.yml](roles/gowave_node/defaults/main.yml).
Override them in inventory or extra variables.

| Variable | Default | Purpose |
| --- | --- | --- |
| `gowave_node_version` | Required | Pinned node release tag, including `v` |
| `gowave_node_core_version` | Required | Pinned Xray release tag, including `v` |
| `gowave_node_secret_key` | Required | Base64 `SECRET_KEY` from the panel |
| `gowave_node_port` | `3000` | TLS management port |
| `gowave_node_install_dir` | `/opt/gowave-node` | Dedicated installation directory under `/opt` or `/srv` |
| `gowave_node_release_repository` | `aaaSaZaN/gowave-node` | Node release repository |
| `gowave_node_core_repository` | `XTLS/Xray-core` | Xray release repository |
| `gowave_node_environment` | `{}` | Additional node environment variables |

For example:

```yaml
gowave_node_environment:
  SNI_VERIFICATION: "true"
  NFTABLES_LOGGING: "false"
  NFTABLES_ACCEPT_REPLY_TRAFFIC: "true"
```

Use quoted strings for booleans. Values must be single-line strings or numbers,
without NUL bytes. The role reserves `SECRET_KEY`, `NODE_PORT`,
`GOWAVE_NODE_RUNTIME_DIR`, `XRAY_CORE_PATH`, `XRAY_LOCATION_ASSET`, and
`ASN_INDEX_PATH` so configuration cannot override its managed settings.
The `.env` renderer follows the node's literal-value parser; do not shell-source
this file.

The systemd unit follows the source project's `install.sh`: it runs as root so
Xray can bind low ports, manage nftables, and access panel-referenced certificates
and shared Unix sockets. Private runtime files live in `/run/gowave-node`, mode
`0700`. Any additional isolation must allow the resources used by your panel
configuration. The optional `/usr/local/bin/geocheck` tool is not installed.

## Updates and operation

Change the pinned versions and rerun the playbook to update. It owns the node
binary, `rw-core`, the `rw-core.stock` reset copy, ASN index, bundled geodata,
complete `.env`, and main `gowave-node.service` unit. Changes to those files
restart the node; unchanged files do not trigger a restart.

The panel and CLI updaters remain available. A subsequent Ansible run restores
the pinned binaries, stock core, and bundled assets. Panel custom-core settings
may reapply a custom core when the panel configures Xray. Avoid concurrent panel
updates, CLI updates, and Ansible deployments. This playbook does not implement
automatic rollback; retain backups before adopting or updating an installation.

This is for fresh or existing native `gowave-node` installations in the selected
directory. Migrate Docker, s6, or legacy `remnanode` installations with the source
project's migration workflow first. Preserve the original installation directory
and review custom systemd overrides before adopting it with Ansible.

The playbook installs `/usr/local/bin/gowave-node` as a CLI link. Run as root:

```sh
sudo gowave-node status
sudo gowave-node logs -n 100 -f
sudo gowave-node xlogs -n 100 -f
sudo journalctl -u gowave-node -n 100
```

Deployment waits up to 30 seconds for the management daemon using
`gowave-node status --wait-running`. Xray starts only after the panel supplies
configuration. After configuring the node in the panel, use
`sudo gowave-node status --wait-ready` to check Xray readiness separately.

`--check` validates inputs and exits before installing packages, downloading
releases, or changing files. It does not predict service startup or health.

## Verify changes

```sh
uv sync --locked
uv run --locked ansible-lint --offline
uv run --locked ansible-playbook -i inventory.example.yml playbook.yml --syntax-check
```

For deployment validation, use a disposable Linux VM with systemd: deploy, rerun
with the same variables, and check that the second run reports no changes.
Confirm `systemctl is-active gowave-node`, connect the panel, and then run
`gowave-node status --wait-ready`. Full deployment and panel/Xray traffic checks
require that environment; syntax and lint checks alone do not validate them.
