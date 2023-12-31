![PolicydRateGuard logo](./docs/logo-dark.png#gh-dark-mode-only)
![PolicydRateGuard logo](./docs/logo-light.png#gh-light-mode-only)

# PolicydRateGuard Role for Ansible

[![Ansible Role onlime.policyd_rate_guard](https://img.shields.io/ansible/role/63151)](https://galaxy.ansible.com/onlime/policyd_rate_guard)
[![Ansible Quality Score](https://img.shields.io/ansible/quality/63151)](https://galaxy.ansible.com/onlime/policyd_rate_guard)
[![Ansible Role Downloads](https://img.shields.io/ansible/role/d/63151)](https://galaxy.ansible.com/onlime/policyd_rate_guard)
[![Latest release](https://img.shields.io/github/v/release/onlime/ansible-role-policyd-rate-guard)](https://github.com/onlime/ansible-role-policyd-rate-guard/releases)

Install and setup [PolicydRateGuard](https://github.com/onlime/policyd-rate-guard) for on a Debian mailserver with Postfix.

## Requirements

The role uses the following Ansible modules: [apt](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html), [pip](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pip_module.html), [git](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/git_module.html), [mysql_db](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_db_module.html), [mysql_user](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_user_module.html) which might have additional dependencies.

It only installs and configures [PolicydRateGuard](https://github.com/onlime/policyd-rate-guard) as a daemon (Systemd service) and will not touch your existing Postfix installation. Please follow the Postfix Configuration instructions in [PolicydRateGuard's README > Production INSTALL](https://github.com/onlime/policyd-rate-guard/blob/main/README.md#configure-postfix).

## Installation

Installation from [Ansible Galaxy `onlime.policyd_rate_guard`](https://galaxy.ansible.com/onlime/policyd_rate_guard):

```bash
$ ansible-galaxy install onlime.policyd_rate_guard
```

Or put the following into your Ansible project's `requirements.yml`:

```yaml
roles:
  - name: onlime.policyd_rate_guard
    version: main
```

... and then run:

```bash
$ ansible-galaxy install -r requirements.yml
```

For usage in your own roles, see **Role Usage Examples** below.

## Role Variables

* `policyd_version`:
  * Default: `main`
  * Description: What version of the repository to check out. This can be the literal string `HEAD`, a branch name, or a tag name.
* `policyd_install_path`:
  * Default: `/opt/policyd-rate-guard`
  * Description: PolicydRateGuard installation directory.
* `policyd_log_path`:
  * Default: `/var/log/policyd-rate-guard`
  * Description: The path to the `policyd-rate-guard.log` log file
* `policyd_sentry_dsn`:
  * Default: `None`
  * Description: Enable Sentry exception reporting by entering your Sentry DSN.
* `policyd_mysql_dbname`:
  * Default: `policyd-rate-guard`
  * Description: MySQL database name.
* `policyd_mysql_user`:
  * Default: `policyd`
  * Description: MySQL database username.
* `policyd_mysql_pass`:
  * Default: `''`
  * Description: MySQL database password.

## Role Usage Examples

Example with minimal customization:

```yaml
- hosts: smtphosts
  roles:
    - role: onlime.policyd_rate_guard
      policyd_version: stable
      policyd_mysql_pass: '{{ vault_policyd_mysql_pass }}'
```

> [!IMPORTANT]
> I recommend to store any credentials like `policyd_mysql_pass` in an [Ansible vault](https://docs.ansible.com/ansible/latest/vault_guide). Best practice is to name all vault variables with a `vault_` prefix, followed by the variable name which you are going to assign it to.

You could also use this Ansible role `onlime.policyd_rate_guard` as a dependency in your own role, using the following structure:

```
roles/myrole/
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
└── vars
    └── main.yml
```

and then put the role's default variables overrides directly into `vars/main.yml`:

```yaml
policyd_version: stable
policyd_mysql_pass: '{{ vault_policyd_mysql_pass }}'
policyd_sentry_dsn: '{{ vault_policyd_sentry_dsn }}'
```

## License

[GPL-3.0 license](LICENSE)

## Authors

This role is maintained by  [Philip Iezzi (Onlime GmbH)](https://www.onlime.ch/).
