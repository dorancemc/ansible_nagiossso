# ansible_nagiossso

OIDC single sign-on for an Apache site, with Auth0 as the broker.

It replaces HTTP Basic auth with federated login (Google, GitHub, Microsoft, SAML
— whatever is enabled in Auth0), and it syncs the allowlist from the inventory
through the Auth0 Management API.

## How it works

`mod_auth_openidc` authenticates against Auth0 and puts the `email` claim into
`REMOTE_USER`. Nagios Core and Thruk both read `REMOTE_USER`, so neither of them
needs to change: the identity arrives on its own, and authorization is still
decided by the Nagios contacts and contactgroups.

The role does not touch the roles that install the web interface. It drops its own
Apache configuration file, `zz-nagiossso.conf`, which loads *after* theirs and
replaces their auth block. Two Apache 2.4 rules make this work:

1. `AuthMerging Off` (the default) makes the authorization directives of a later
   section **replace** the earlier ones instead of adding to them.
2. `<DirectoryMatch>` sections are applied in configuration order, and always
   after the plain `<Directory>` sections.

## What it protects

`nagiossso_protected_paths` is a list of **filesystem paths**, not URLs. They
become the alternatives of one `<DirectoryMatch>` regex. Adding an application
means adding a path. The role ships the Nagios CGI paths as the default, but the
inventory decides what is really protected.

This suits applications that read `REMOTE_USER` and take their authorization from
Nagios. It does not suit applications that need their own claims and run their own
OAuth flow.

Keep these **out** of the list:

- **NRDP** (`/usr/local/nrdp`): machine to machine, authenticated with a token.
  OIDC would break every passive sender.
- **Thruk's API** (`/thruk/cgi-bin/remote.cgi` and `/thruk/r/`): also token based.
  With OIDC on top, `thruk backend list` and the maintenance cron job stop working.
- **Any app behind the same Apache that runs its own OAuth**, such as Grafana.

This is why `AuthType` lives in a `<DirectoryMatch>` over concrete paths, and never
at vhost level or in `<Location />`. Widening it to the site root breaks those apps.

## Break-glass

With `nagiossso_breakglass_enabled` (default `true`), requests from `127.0.0.1`
keep using Basic auth against the existing htpasswd file. It is reachable only
through a tunnel:

```bash
ssh -L 8443:127.0.0.1:443 <host>
```

Then open `https://127.0.0.1:8443/`. It is real HTTPS against port 443 of the
server, so `SSLRequireSSL` is still satisfied; only `REMOTE_ADDR` changes.

**Test it before you trust the SSO.** If Auth0 is down, or the configuration is
wrong, this is the only way in.

## The allowlist

It is derived from the Nagios contacts flagged with `sso: true`, so the inventory
is the single source of truth:

```yaml
nagiosconfig_contacts:
  soporte@example.com:
    alias: Soporte Example
    use: none-notifications
    contactgroups: example.com   # what they see
    sso: true                    # they may log in
```

`tasks/auth0.yml` turns that into the code of an Auth0 post-login Action, deploys
it, and binds it to the trigger. It needs `ansible_nagiosconfig` to drop the `sso`
key when it writes `contacts.cfg`.

Outside this project, pass `nagiossso_allowed_emails` as a flat list instead.

## Things that break quietly

- **The `zz-` prefix** of the Apache configuration file. It is what makes the file
  load last. Rename it and Basic auth takes over again, with no error and no
  warning. Check with `curl -sI https://<host>/<path>/`: it must answer `302`,
  never `401 WWW-Authenticate: Basic`.
- **The handler name** `Reload nagiossso web server` carries the role prefix on
  purpose. The `certbot` role has a handler called `Reload web server` and flushes
  handlers before it asks for the certificate; a handler with the same name would
  fire there, at a point where the Apache configuration may not be valid yet.
- **The allowlist lives in the Action code**, not in a secret. Auth0 secrets are
  write only: a `GET` never returns the value, so changes could not be detected and
  every run would report `changed`.
- **A `PATCH` without the `deploy` call changes no login.** The Action stays as a
  draft, and the console makes it look applied.
- **The bindings `PATCH` replaces the whole list** of the trigger, so the role
  rebuilds it in full. Sending only its own binding would unbind every other Action
  in the tenant.
- **The Action runs for every application in the tenant.** It opens with a guard on
  `client_id` so it does not deny the logins of other apps.
- **An empty allowlist revokes everyone's access**, in green and with no warning.
  The usual cause is running the Auth0 sync without the role that loads the Nagios
  contacts, so the role refuses to deploy one. Set
  `nagiossso_allow_empty_allowlist: true` only when access is meant to depend on
  `nagiossso_allowed_domains` alone.

## Variables

See `defaults/main.yml`. Secrets go in a `vault.yaml` under `group_vars`, encrypted
value by value.

## Tags

| Tag | What it runs |
|---|---|
| `nagiossso` | The whole role |
| `nagiossso-install` | Package and Apache module |
| `nagiossso-config` | Configuration file plus `apachectl -t` |

`tasks/auth0.yml` has no tags of its own. It is not imported from `tasks/main.yml`:
the playbook calls it with `tasks_from: auth0` and applies the tags itself.

## Order in the playbook

```yaml
roles:
  - { role: nagiossso, when: nagiossso_enabled | default(false) | bool }
  - role: nagioscore
post_tasks:
  - name: Sync the Auth0 allowlist
    ansible.builtin.include_role:
      name: nagiossso
      tasks_from: auth0
      apply:
        tags:
          - nagiossso
          - nagiossso-auth0
    when: nagiossso_manage_action | default(false) | bool
    tags:
      - nagiossso
      - nagiossso-auth0
```

- **Before `nagioscore`**: `libapache2-mod-auth-openidc` pulls in `apache2`, so the
  module is loaded before a vhost with `AuthType openid-connect` exists. The other
  way round, Apache does not start.
- **`auth0` in `post_tasks`**: it needs `nagiosconfig_contacts`, which only exists
  after `nagiosconfig` has read the inventory.
- **The `apply` block is not optional.** In a dynamic `include_role` the tags of the
  include are not inherited by the included tasks, and `auth0.yml` brings none. Without
  it the task reports "included" and runs nothing, with the play green.

## Manual bootstrap in Auth0

Two applications, created once. You cannot call the Management API without
credentials for the Management API.

| Application | Type | Purpose |
|---|---|---|
| Web interface | Regular Web Application | OIDC client. Callback: `https://<host>/<path>/redirect_uri` |
| ansible-nagios | Machine to Machine | Manages the Action. Scopes: `read:actions`, `create:actions`, `update:actions` |

## Debugging

`nagiossso_auth0_no_log: false` shows the API calls. It leaves traces of the bearer
token in the output: use it only for a short time, and never in CI.

## Requirements

- Ansible 2.20 or newer.
- Debian 13 or Enterprise Linux 9, with Apache already installed.
- An Auth0 tenant.

## License

Apache-2.0
