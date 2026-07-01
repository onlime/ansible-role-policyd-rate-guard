# CHANGELOG

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
