# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-08-23

First release.

### Added

- OIDC authentication for the Nagios Core web interface through `mod_auth_openidc`, with Auth0 as the broker. `REMOTE_USER` carries the `email` claim, which the Nagios CGIs already consume.
- `zz-nagiossso.conf`: a standalone Apache configuration that overrides the Basic auth of `ansible_nagioscore` without modifying that role. Relies on `AuthMerging Off` and on `<DirectoryMatch>` sections being applied in configuration order.
- Break-glass path (`nagiossso_breakglass_enabled`): requests from `127.0.0.1` keep Basic auth against the existing htpasswd, reachable over an SSH tunnel when the identity provider is unavailable.
- `tasks/auth0.yml`: renders, creates or updates, deploys and binds an Auth0 post-login Action holding the allowlist. Idempotent by comparing the deployed action code; skips the write when it matches and is already deployed.
- Allowlist derived from the Nagios contacts flagged with `sso: true`, so the inventory is the single source of truth. `nagiossso_allowed_emails` accepts a flat list for standalone use.
- `nagiossso_allowed_domains`: whole domains authenticate without a nominal entry; scope is still decided by the contact's contactgroups.
- Cross-check assert: fails when an allowlisted address has no matching Nagios contact, which otherwise shows up as "logs in but sees nothing".
- `apachectl -t` before the reload handler, so a broken auth configuration stops the play instead of locking everyone out of the monitoring server.
