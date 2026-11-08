![PolicydRateGuard logo](./docs/logo-dark.png#gh-dark-mode-only)
![PolicydRateGuard logo](./docs/logo-light.png#gh-light-mode-only)

# PolicydRateGuard Role for Ansible

[![CI](https://github.com/onlime/ansible-role-policyd-rate-guard/actions/workflows/ci.yml/badge.svg)](https://github.com/onlime/ansible-role-policyd-rate-guard/actions/workflows/ci.yml)
[![Ansible Role onlime.policyd_rate_guard](https://img.shields.io/ansible/role/d/onlime/policyd_rate_guard)](https://galaxy.ansible.com/onlime/policyd_rate_guard)
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
- Ansible 2.16 or later (ansible-core) and the collection `ansible.mysql` on the controller; the role installs `python3-pymysql` on the host for it. The `mysql_db`/`mysql_user` tasks log in as configured for root, e.g. in `/root/.my.cnf`, or through `policyd_mysql_login_unix_socket`.

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
| `policyd_mysql_users` | the daemon's user | Users and grants, extend it for read-only users etc. (the migration user is managed separately) |
| `policyd_mysql_dbname` | `policyd-rate-guard` | Database name |
| `policyd_mysql_unix_socket` | `/run/mysqld/mysqld.sock` | Socket the daemon and the migrations connect through (empty: TCP `localhost:3306`) |
| `policyd_mysql_login_unix_socket` | | Socket for the `mysql_db`/`mysql_user` tasks |
| `policyd_mysql_plugin` | `caching_sha2_password` | Authentication plugin of the users (`''` for the server default) |
| `policyd_rotate_passwords` | `false` | Update the passwords of existing users |
| `policyd_listen` | `['127.0.0.1:10033']` | Listen addresses of the socket unit; a path like `/var/spool/postfix/policyd-rate-guard/policyd.sock` (Postfix: `unix:policyd-rate-guard/policyd.sock`) creates a unix socket owned by `policyd_socket_user`/`_group` (`postfix`) with `policyd_socket_mode` (`0600`). Only in a directory nobody but root can write to: the role refuses one like Postfix's `private/`, because systemd's chown of the socket follows a symlink. systemd creates a missing directory as root 0755. |
| `policyd_quota` | `1000` | Default quota of new senders |
| `policyd_action_text_blocked` | | Text of the answer for senders over quota |
| `policyd_fail_action` | `DUNNO` | Answer on internal errors: `DUNNO` or `DEFER_IF_PERMIT` |
| `policyd_message_retention` | `365` | Days to keep rows in `messages` (`0`: forever) |
| `policyd_log_level`, `policyd_log_rotate`, `policyd_syslog` | `INFO`, `365`, `true` | Logging to `/var/log/policyd-rate-guard/policyd-rate-guard.log` (rotated daily) and syslog |
| `policyd_sentry_dsn`, `policyd_sentry_environment` | , `prod` | Sentry error reporting |
| `policyd_webhook_enabled`, `policyd_webhook_url`, `policyd_webhook_secret` | `false`, , `vault_policyd_webhook_secret` | Webhook when a sender reaches the quota, authenticated with a JWT. The secret is base64 and decodes to at least 32 bytes: `openssl rand -base64 32` |
| `policyd_extra_env` | `{}` | More environment variables, e.g. `{DB_POOL_MAXCONNECTIONS: 20}` |
| `policyd_user`, `policyd_install_path`, `policyd_config_path` | `policyd-rate-guard`, `/opt/policyd-rate-guard`, `/etc/policyd-rate-guard` | Paths and user |

## Upgrading from 0.x

Role 1.0 installs PolicydRateGuard 1.x; upgrade both together. It doesn't upgrade a 0.x installation in place: role 0.5.2 and later made the installation directory owned by `postfix` and ran code from it as root, so nothing in it can be trusted. Role 1.0 refuses an installation directory that isn't owned by root throughout, or that still has 0.x's `run.py`, and changes nothing. Move the old installation away, then the role installs 1.0 fresh. The database schema is unchanged, so the data stays as it is.

1. **Update your variables:**
   - Add `policyd_yoyo_mysql_pass`: the migrations use their own database user now, and the daemon's user loses its schema privileges.
   - Pin `policyd_version` to a tag if you set it to a branch.
   - Check `policyd_message_retention`: it's 365 days now, 0.x kept everything.
   - With the webhook enabled: `policyd_webhook_secret` must be base64 and decode to at least 32 bytes (`openssl rand -base64 32`). Remove `policyd_webhook_use_jwt` and a `{token}` in `policyd_webhook_url`: PolicydRateGuard 1.0 always sends a JWT.
   - Remove `policyd_log_path`: the log is always in `/var/log/policyd-rate-guard/`, and `policyd_log_file` must stay there.

   The role checks these before changing anything.
2. **Move the 0.x installation away**, root only, and restore the mode of `/etc/systemd/system` (role 0.x set it to `0644`):
   ```bash
   $ mv /opt/policyd-rate-guard /opt/policyd-rate-guard.pre-1.0
   $ chown root:root /opt/policyd-rate-guard.pre-1.0
   $ chmod 0700 /opt/policyd-rate-guard.pre-1.0
   $ chmod 0755 /etc/systemd/system
   ```
   The 0.x daemon keeps serving from memory until the role replaces its unit at the end of its run. Run the role right away: 0.x's logrotate restarts the daemon every night, and then it no longer starts.
3. **Run the role:** with `--check --diff` first (it shows a fresh installation), then for real, then once more, which must report no changes.
4. **Check the result:**
   ```bash
   $ systemctl status policyd-rate-guard.socket policyd-rate-guard.service policyd-rate-guard-cleanup.timer
   $ journalctl -u policyd-rate-guard
   ```
   Then send a mail through Postfix with an authenticated account, and look for its line in `/var/log/policyd-rate-guard/policyd-rate-guard.log`.
5. **Rotate the database passwords and the webhook secret:** 0.x deployed them in world-readable files. Set new values and run the role with `-e policyd_rotate_passwords=true`. Change the webhook secret on the receiving side at the same time.
6. **Once 1.0 runs fine,** delete `/opt/policyd-rate-guard.pre-1.0`.

**Back to 0.x**, as long as the backup exists:
1. Stop and disable the 1.0 units, and remove them: `policyd-rate-guard.socket`, `policyd-rate-guard.service`, `policyd-rate-guard-cleanup.service` and `policyd-rate-guard-cleanup.timer` in `/etc/systemd/system`.
2. Move `/opt/policyd-rate-guard` away and the backup back, and give it back to `postfix`.
3. Install 0.x's units and logrotate file from its `deployment/` directory, and start its service and timer.

After a password rotation, put the current database password into the restored `.env` first.

## Development

The role is tested in two ways, locally and by [GitHub Actions](https://github.com/onlime/ansible-role-policyd-rate-guard/actions/workflows/ci.yml) on pushes to `main` and `develop` and on pull requests:

- **ansible-lint** checks the code against Ansible's best practices (profile `production`).
- **Molecule** runs the role in a Docker container with Debian 13, systemd, Postfix and Percona Server 8.4, like a real mail server, and then checks the result. The container is created for the test and removed afterwards; on your machine, only the tools in `.venv` (see below) and the Docker image remain.

You need no knowledge of Molecule or Docker: the commands below are all there is to it.

### Requirements

- [Docker](https://docs.docker.com/get-started/get-docker/), running: Docker Desktop on macOS and Windows, Docker Engine on Linux. The test container runs systemd, so Molecule starts it with `--privileged`.
- Python 3.13 or later, and git.
- Internet access: the container downloads Debian packages, Percona Server, PolicydRateGuard and its dependencies. The Docker image needs about 1.1 GB of disk space.

### Setup (once)

```bash
$ git clone https://github.com/onlime/ansible-role-policyd-rate-guard.git
$ cd ansible-role-policyd-rate-guard
$ python3 -m venv .venv
$ .venv/bin/pip install ansible-core ansible-lint molecule 'molecule-plugins[docker]'
```

The tools are installed in `.venv` in the repository (ignored by git). Activate it in every new shell before running the commands below:

```bash
$ source .venv/bin/activate
```

### Lint

```bash
$ ansible-lint --profile production
```

The first run installs the collection `ansible.mysql` (from `requirements.yml`). It ends with `Passed: 0 failure(s), 0 warning(s)`.

### Tests

The test takes about three minutes; the first run also downloads the Docker image. It's a fresh install: a dry run (`--check`) first, the install, a second run that must change nothing, another dry run, then the checks. They include: permissions and ownership of all files, the units (socket, service, cleanup timer), the database migrations and privileges, a real policy request that is answered and counted, the log line format, log rotation, the nightly cleanup and its retry, a restart that keeps the socket open, and the systemd sandbox score. Also tested: a unix socket for Postfix, the webhook configuration, a module planted in `/tmp` that root must never import, and that the role refuses an installation owned by `postfix` (as 0.x left it).

```bash
$ molecule test
```

The test passed when the command ends without an error and the `SCENARIO RECAP` at the end shows `failed=0`. By default it installs the PolicydRateGuard release the role is made for (`policyd_version` in `defaults/main.yml`) from GitHub.

To test against a local checkout of PolicydRateGuard instead, e.g. a branch you're working on, clone it next to the role:

```bash
$ git clone https://github.com/onlime/policyd-rate-guard.git ../policyd-rate-guard
$ POLICYD_REPO_PATH=$PWD/../policyd-rate-guard POLICYD_VERSION=main molecule test
```

`POLICYD_REPO_PATH` must be an absolute path. `POLICYD_VERSION` is a local branch (e.g. the one you're working on), a tag or a commit of that checkout. The role clones the checkout, so only committed changes are tested, and a branch that only exists on GitHub must be checked out locally first (`git -C ../policyd-rate-guard switch develop`).

### When a test fails

`molecule test` removes the container at the end, also after a failure. To look at what happened, run the steps one by one, which keeps the container:

```bash
# create and prepare the container, run the role:
$ molecule converge
# run the checks again:
$ molecule verify
# a shell in the container, e.g. for: systemctl status policyd-rate-guard
$ molecule login
# remove the container when you're done:
$ molecule destroy
```

After you change the role, `molecule converge` runs it again on the same container. `molecule destroy` also cleans up after an interrupted run (Ctrl-C); `molecule test` starts with a clean container anyway.

## License

[GPL-3.0 license](LICENSE)

## Authors

This role is maintained by [Philip Iezzi (Onlime GmbH)](https://www.onlime.ch/).
