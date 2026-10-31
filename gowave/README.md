# GoWave with Ansible

This playbook installs a pinned GoWave release on Linux x86_64 or aarch64 hosts
running systemd. It downloads both the backend and frontend, verifies them against
the release's `SHA256SUMS`, configures the `gowave` service account, and waits for
`/api/system/health` to return HTTP 200. No Go, Bun, Docker, or Redis is needed on
the target.

The target needs Python 3, SSH access, privilege escalation to root, working
package repositories, and HTTPS access to GitHub release assets. Use a dedicated
installation directory under `/opt` or `/srv`. The default is `/opt/gowave`.

## Deploy

Run these commands from this directory. Install [uv](https://docs.astral.sh/uv/)
on the controller first.

```sh
uv sync --locked
cp inventory.example.yml inventory.local.yml
```

Edit `inventory.local.yml`: set the SSH host/user, public domains, and
`gowave_version` to an existing `vX.Y.Z` release tag. Prerelease tags are also
accepted. The version is required; the playbook does not resolve `latest` or `dev`.

Generate a secret with `openssl rand -hex 32`, then create an encrypted variables
file:

```sh
uv run --locked ansible-vault create secrets.local.yml
```

Enter the generated value in that file:

```yaml
vault_gowave_app_secret: YOUR_GENERATED_SECRET
```

Keep this secret stable across deploys. The example inventory references it through
`gowave_app_secret`. Configuration tasks hide secrets from logs and diffs; the
remote `.env` is owned by `root:gowave` with mode `0640`.

```sh
uv run --locked ansible-playbook -i inventory.local.yml playbook.yml \
  -e @secrets.local.yml --ask-vault-pass --ask-become-pass
```

Omit `--ask-become-pass` when your SSH account has passwordless sudo. For multiple
panels, add hosts to the `gowave` group and use host variables for their domains,
database connections, and secrets. Local inventory/secret files ending in
`.local.yml` are ignored by Git.

## Configuration

Defaults are in [roles/gowave/defaults/main.yml](roles/gowave/defaults/main.yml).
Override them in inventory or with extra variables.

| Variable | Default | Purpose |
| --- | --- | --- |
| `gowave_version` | Required | Release tag, including the `v` prefix |
| `gowave_release_repository` | `aaaSaZaN/gowave` | GitHub owner/repository |
| `gowave_install_dir` | `/opt/gowave` | Dedicated native installation directory |
| `gowave_app_secret` | Required | Private random alphanumeric secret, at least 32 characters |
| `gowave_app_port` | `3000` | Loopback HTTP port |
| `gowave_panel_domain` | `localhost:3000` | Public dashboard hostname |
| `gowave_panel_base_path` | Empty | Optional dashboard path; API/subscription paths stay unchanged |
| `gowave_front_end_domain` | `*` | Allowed frontend origin |
| `gowave_sub_public_domain` | `localhost:3000/api/sub` | Public subscription domain/path, without scheme |
| `gowave_db_driver` | `sqlite` | `sqlite` or `postgres` |
| `gowave_database_url` | `gowave.db` | SQLite filename or existing PostgreSQL connection URL |
| `gowave_environment` | `{}` | Additional environment variables |

SQLite data lives in the installation directory by default and is retained on
repeat deployments. For an existing PostgreSQL database, set both
`gowave_db_driver: postgres` and `gowave_database_url` to its connection URL. Put
database credentials in the encrypted variables file. Database provisioning and
database-engine migrations are separate operations.

Metrics are disabled by default. Extra settings may override the rendered
environment defaults, except `APP_SECRET`, `APP_HOST`, and `APP_PORT`. Use quoted
strings for booleans and JSON arrays, for example:

```yaml
gowave_environment:
  METRICS_ENABLED: "true"
  METRICS_PORT: "3001"
  METRICS_USER: monitoring
  METRICS_PASS: "{{ vault_gowave_metrics_password }}"
  IS_HTTP_LOGGING_ENABLED: "false"
```

The application binds to `127.0.0.1`. Configure your HTTPS reverse proxy on the
same host to forward the complete request path to `127.0.0.1:3000` (or your chosen
port), including WebSocket upgrades. TLS certificates, DNS, firewall rules, and
the separate subscription-page application are managed outside this playbook.

## Updates and operation

Change `gowave_version` and rerun the playbook to update. Back up your database
first: startup applies migrations, so deploying an older binary does not reverse
schema changes. The playbook manages the complete `.env`; put custom settings in
inventory rather than editing the remote file.

The main systemd unit includes the hardening policy from
`backend-go/internal/serviceunit/install.sh`. The playbook also calls the project's
`gowave service install` command when the binary changes or an upstream unit is
missing. This maintains the release's hardening drop-in and update watcher. The
application runs as `gowave`, while application code and frontend files remain
root-owned. `/usr/local/bin/gowave` links to the installed binary. Use
`journalctl -u gowave -n 100` to inspect startup failures.

The in-panel updater remains available and can install another version. Rerunning
Ansible restores the pinned backend/frontend. Avoid running the updater and this
playbook concurrently. Back up and review configuration before adopting an
existing installation: its `.env` and main systemd unit will be replaced.

`--check` validates inputs only and exits before making changes or downloading
releases. It does not predict extraction, service installation, or health.

## Verify changes

```sh
uv sync --locked
uv run --locked ansible-lint --offline
uv run --locked ansible-playbook -i inventory.example.yml playbook.yml --syntax-check
```

For a deployment check, use a disposable Linux VM with systemd: deploy once,
rerun with the same variables, and confirm the second run reports no changes.
Check that the frontend loads, `systemctl is-active gowave gowave-update.path`
succeeds, and the `gowave` account cannot modify `gowave-server` or frontend files.

The playbook uses only built-in Ansible modules; release downloads use
[`get_url` checksum verification](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/get_url_module.html).
