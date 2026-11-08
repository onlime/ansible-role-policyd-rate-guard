# CHANGELOG

## [v1.0.0](https://github.com/onlime/ansible-role-policyd-rate-guard/tree/v1.0.0) (unreleased)

For [PolicydRateGuard v1.0.0](https://github.com/onlime/policyd-rate-guard/releases/tag/v1.0.0), Debian 13 only. See "Upgrading from 0.x" in the README.

**Security:**

- The configuration with the database password and the webhook secret was world-readable (`.env` and `yoyo.ini` with mode `0644` in the code directory). It's now in `/etc/policyd-rate-guard/`: the env file `root:policyd-rate-guard 0640`, `yoyo.ini` `root:root 0600`.
- Since v0.5.2 the code directory, including `.git` and the venv, was owned by `postfix`, and the role ran code from it as root (yoyo migrations, pip, git). Anything running as `postfix`, like Postfix's smtpd, could have planted code there that root would run on the next deployment. The code is now owned by root and never writable by the service; an existing installation not owned by root is moved aside instead of reused.
- The daemon runs as a dedicated user `policyd-rate-guard` instead of `postfix` (the mail owner), with systemd sandboxing.
- `policyd_version` defaults to a release tag instead of `main`, so a run no longer deploys whatever is on the default branch.
- PolicydRateGuard is installed with `pip install --only-binary=:all:` of the checkout: dependencies as binary wheels only, so no third-party code runs at install time, and only when the checked-out commit changes. pip is no longer upgraded to the latest version.
- The migrations use their own database user; the daemon's user only gets `SELECT, INSERT, UPDATE, DELETE`.

**Changed (breaking):**

- Debian 13 (trixie) only.
- New required variable `policyd_yoyo_mysql_pass`.
- `policyd_message_retention` defaults to `365` (was `0`, keep forever).
- Systemd units are templated by the role (socket, service, cleanup service and timer) instead of copied from the checkout; `policyd_listen` configures the socket.
- Renamed handler and tasks; `quick-deploy` now also covers migrations and units.
- `policyd_log_path` is removed: the log is always in `/var/log/<service name>/`, which the unit creates (`LogsDirectory=`) and which is the only place the sandboxed daemon may write to. `policyd_log_file` must stay in that directory; logrotate now creates the files with the service group instead of `adm`.
- `policyd_webhook_use_jwt` is removed: PolicydRateGuard 1.0 always authenticates the webhook with a JWT in the `Authorization` header (the simple token is gone). The role fails early if it's still set to `false`.

**Improved:**

- The database users' passwords are only changed with `policyd_rotate_passwords=true`. Before, the user task reported a change on every run (`caching_sha2_password` hashes can't be compared), reset the passwords and restarted the daemon.
- The env file is validated with `policyd-rate-guard check` (configuration and database connection) before it replaces the old one.
- The venv is recreated when Python's minor version changes (Debian release upgrade).
- Migrations are only applied when some are pending, so the role reports changes correctly.
- The daemon is only restarted when code, dependencies, configuration or units change; logrotate no longer restarts it.
- Works in check mode, including on a 0.x host.

**Fixed:**

- Since v0.5.2 every run after the first failed at the git task: git refuses a repository owned by another user (`postfix`) when run as root ("dubious ownership").
- The first run on a fresh host failed: the first migration ran before the database users were created. The migration user is now created first, the other users and their grants (which may refer to tables) after the migrations.
- Passwords with special characters broke `yoyo.ini` (`%` and URL-reserved characters weren't escaped); values in the env file are quoted.
- The service was started before systemd was reloaded, so changed unit files only took effect on the next restart.
- Molecule tests (fresh install and upgrade from 0.5.3/0.7.4 on Debian 13 with Percona Server 8.4) and GitHub Actions.

## [v0.5.3](https://github.com/onlime/ansible-role-policyd-rate-guard/tree/v0.5.3) (2026-07-01)

- Modernize role to pass `ansible-lint` at the `production` profile
- Migrate MySQL tasks to the new `ansible.mysql` collection (`ansible.mysql.mysql_db`, `ansible.mysql.mysql_user`), replacing the now-deprecated `community.mysql` modules (removed in `community.mysql` 6.0.0)
- Rename handlers to satisfy `name[casing]` (`Apply yoyo migrations`, `Reload systemd daemon`, `Restart policyd service`)
- Modernize task syntax: `with_items` → `loop`, `when: policyd_db.changed` → `when: policyd_db is changed`
- Set `namespace: onlime` and bump `min_ansible_version` to `2.15` in `meta/main.yml`

## [v0.5.2](https://github.com/onlime/ansible-role-policyd-rate-guard/tree/v0.5.2) (2026-02-24)

- Fix ownership of install directory to ensure it is owned by the `postfix` user and group, as it's recommended to run the Systemd `policyd-rate-guard.service` as `postfix` user for security reasons.

## [v0.5.1](https://github.com/onlime/ansible-role-policyd-rate-guard/tree/v0.5.1) (2025-02-27)

- Support `policyd_webhook_use_jwt` (bool) variable for enabling JWT authentication for the `WEBHOOK_USE_JWT` env var which was introduced in [onlime/policyd-rate-guard v0.7.2](https://github.com/onlime/policyd-rate-guard/releases/tag/v0.7.2)
- Migrate the `policyd` MySQL user to the `caching_sha2_password` plugin
- Add `.env` vars for the new webhook introduced in [onlime/policyd-rate-guard v0.7.0](https://github.com/onlime/policyd-rate-guard/releases/tag/v0.7.0)
- Add `policyd_message_retention` variable for the `MESSAGE_RETENTION` env var introduced in [onlime/policyd-rate-guard v0.6.1](https://github.com/onlime/policyd-rate-guard/releases/tag/v0.6.1)
- Disable `LOG_MSG_PREFIX` (new in [onlime/policyd-rate-guard v0.5.1](https://github.com/onlime/policyd-rate-guard/releases/tag/v0.5.1)) in the recommended production `.env`
- Add installation instructions to the README

## [v0.5.0](https://github.com/onlime/ansible-role-policyd-rate-guard/tree/v0.5.0) (2023-08-29)

- Initial release
