![PolicydRateGuard logo](./docs/logo-dark.png#gh-dark-mode-only)
![PolicydRateGuard logo](./docs/logo-light.png#gh-light-mode-only)

# PolicydRateGuard Role for Ansible

[![CI](https://github.com/onlime/ansible-role-policyd-rate-guard/actions/workflows/ci.yml/badge.svg)](https://github.com/onlime/ansible-role-policyd-rate-guard/actions/workflows/ci.yml)
[![Ansible Role onlime.policyd_rate_guard](https://img.shields.io/ansible/role/63151)](https://galaxy.ansible.com/onlime/policyd_rate_guard)
[![Latest release](https://img.shields.io/github/v/release/onlime/ansible-role-policyd-rate-guard)](https://github.com/onlime/ansible-role-policyd-rate-guard/releases)

Installs [PolicydRateGuard](https://github.com/onlime/policyd-rate-guard), a sender rate limit policy daemon for Postfix, on a Debian 13 mail server.

What the role sets up:

- A dedicated system user `policyd-rate-guard` that runs the daemon and the daily cleanup job.
- The code in `/opt/policyd-rate-guard`, checked out at a release tag and **owned by root**, installed into a venv with `pip install` (dependencies as binary wheels only).
- The configuration in `/etc/policyd-rate-guard/`: the env file is readable by root and the service user only, `yoyo.ini` (credentials of the migration user) by root only.
- The database, two database users (the daemon's with data access only, one for the schema migrations) and the migrations.
- Hardened systemd units: a socket unit (`127.0.0.1:10033` by default), the service with `Type=notify` and sandboxing, and the cleanup service and timer. Log rotation without restarts.

It doesn't touch your Postfix configuration; see [Configure Postfix](https://github.com/onlime/policyd-rate-guard#configure-postfix) in PolicydRateGuard's README.

## Requirements

- Debian 13 (trixie) or later, with MySQL/Percona Server 8.4+ or MariaDB 12.1+ on the same host (for older MariaDB set `policyd_mysql_plugin: ''`).
- The collection `ansible.mysql` on the controller; the role installs `python3-pymysql` on the host for it. The `mysql_db`/`mysql_user` tasks log in as configured for root, e.g. in `/root/.my.cnf`, or through `policyd_mysql_login_unix_socket`.

## Installation

```bash
$ ansible-galaxy role install onlime.policyd_rate_guard
```

or in your `requirements.yml`:

```yaml
roles:
  - name: onlime.policyd_rate_guard
    version: v1.0.0
```

## Usage

```yaml
- hosts: smtphosts
  roles:
    - role: onlime.policyd_rate_guard
      policyd_mysql_pass: '{{ vault_policyd_mysql_pass }}'
      policyd_yoyo_mysql_pass: '{{ vault_policyd_yoyo_mysql_pass }}'
```

Keep the passwords in an [Ansible vault](https://docs.ansible.com/ansible/latest/vault_guide/). Tags: `quick-deploy` (code, venv, migrations and units only), `policyd-config`, `policyd-mysql-users`, `policyd-logrotate`.

To change the database passwords, update them in the vault and run once with `-e policyd_rotate_passwords=true`. Without it, the passwords of existing users are left alone: the module can't compare `caching_sha2_password` hashes and would otherwise reset them on every run.

## Role variables

| Variable | Default | Description |
| --- | --- | --- |
| `policyd_version` | `v1.0.0` | Release tag (or commit) of PolicydRateGuard. Don't use a branch: every run would deploy whatever it points to. |
| `policyd_repo` | GitHub | Git repository |
| `policyd_mysql_pass` | `vault_policyd_mysql_pass` | Password of the daemon's database user `policyd_mysql_user` (`policyd`) |
| `policyd_yoyo_mysql_pass` | `vault_policyd_yoyo_mysql_pass` | Password of the migration user `policyd_yoyo_mysql_user` (`policyd-rate-guard`) |
| `policyd_mysql_users` | both users | Users and grants, extend it for read-only users etc. |
| `policyd_mysql_dbname` | `policyd-rate-guard` | Database name |
| `policyd_mysql_unix_socket` | `/run/mysqld/mysqld.sock` | Socket the daemon and the migrations connect through (empty: TCP `localhost:3306`) |
| `policyd_mysql_login_unix_socket` | | Socket for the `mysql_db`/`mysql_user` tasks |
| `policyd_mysql_plugin` | `caching_sha2_password` | Authentication plugin of the users (`''` for the server default) |
| `policyd_rotate_passwords` | `false` | Update the passwords of existing users |
| `policyd_listen` | `['127.0.0.1:10033']` | Listen addresses of the socket unit; a path like `/var/spool/postfix/private/policyd-rate-guard` creates a unix socket owned by `policyd_socket_user`/`_group` (`postfix`) with `policyd_socket_mode` (`0600`) |
| `policyd_quota` | `1000` | Default quota of new senders |
| `policyd_action_text_blocked` | | Text of the answer for senders over quota |
| `policyd_fail_action` | `DUNNO` | Answer on internal errors: `DUNNO` or `DEFER_IF_PERMIT` |
| `policyd_message_retention` | `365` | Days to keep rows in `messages` (`0`: forever) |
| `policyd_log_level`, `policyd_log_file`, `policyd_log_rotate`, `policyd_syslog` | `INFO`, `/var/log/policyd-rate-guard/policyd-rate-guard.log`, `365`, `true` | Logging |
| `policyd_sentry_dsn`, `policyd_sentry_environment` | , `prod` | Sentry error reporting |
| `policyd_webhook_enabled`, `policyd_webhook_url`, `policyd_webhook_secret`, `policyd_webhook_use_jwt` | `false`, , `vault_policyd_webhook_secret`, `false` | Webhook when a sender reaches the quota; use JWT |
| `policyd_extra_env` | `{}` | More environment variables, e.g. `{DB_POOL_MAXCONNECTIONS: 20}` |
| `policyd_user`, `policyd_install_path`, `policyd_config_path` | `policyd-rate-guard`, `/opt/policyd-rate-guard`, `/etc/policyd-rate-guard` | Paths and user |

## Upgrading from 0.x

Role 1.0 installs PolicydRateGuard 1.x; upgrade both together. Role 0.5.2 and later made the installation directory owned by `postfix` and ran code from it as root, so the role doesn't reuse it: an installation directory not owned by root is moved to `/opt/policyd-rate-guard.pre-1.0` (root only, for inspection; delete it afterwards), then a fresh clone and venv are created. The old service keeps running until the new socket unit takes over port 10033, near the end of the run.

You need to:

- Add `policyd_yoyo_mysql_pass` (the migrations now use their own database user; the daemon's user loses its schema privileges).
- Pin `policyd_version` to a tag if you set it to a branch.
- Check `policyd_message_retention` (now 365 days instead of keeping everything).
- Rotate the database passwords and the webhook secret afterwards: 0.x deployed them in world-readable files.

Run the role in check mode (`--check --diff`) first; on a 0.x host it reports the move and the new installation.

## Development

```bash
$ ansible-lint --profile production
# fresh install and upgrade from 0.5.3/0.7.4, on Debian 13 with Percona Server 8.4 (needs Docker)
$ molecule test -s default
$ molecule test -s upgrade
# against a local checkout of PolicydRateGuard
$ POLICYD_REPO_PATH=../policyd-rate-guard POLICYD_VERSION=develop molecule test -s upgrade
```

## License

[GPL-3.0 license](LICENSE)

## Authors

This role is maintained by [Philip Iezzi (Onlime GmbH)](https://www.onlime.ch/).
