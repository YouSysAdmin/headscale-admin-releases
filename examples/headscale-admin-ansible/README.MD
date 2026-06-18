# Ansible role: headscale-admin

Installs and configures **headscale-admin** - the single-binary admin UI that
also supervises a *managed* headscale.

headscale-admin downloads the pinned headscale release itself (SHA-256
verified) and then generates and owns `headscale.yaml` + the sqlite DB / keys
under its `work_dir`. It is the only public listener: it terminates TLS and
proxies `/admin/*` to the UI and everything else to headscale.

Because of that, this role:

1. **Installs the headscale-admin binary** - from GitHub releases, a direct
   URL, or S3.
2. **Writes headscale-admin's config** - `config.yaml` for non-secret settings
   plus an env file for `HSA_*` secrets (the same two config methods
   headscale-admin supports).
3. **Optionally pre-configures the managed headscale** - *config only*. The
   headscale binary is still downloaded and supervised by headscale-admin.

## Layout created

| Path                                   | Purpose                                   |
| -------------------------------------- | ----------------------------------------- |
| `headscale-admin:headscale-admin`      | system user / group                       |
| `/opt/headscale-admin/`                | home - holds `config.yaml`, `env`, `bin/`, `data/` |
| `/opt/headscale-admin/bin/`            | `headscale.bin_dir`: the headscale-admin binary plus the downloaded headscale / headscale-pf releases |
| `/opt/headscale-admin/bin/headscale-admin` | the binary (root:group, 0755)         |
| `/opt/headscale-admin/config.yaml`     | non-secret config (root:group, 0640)      |
| `/opt/headscale-admin/env`             | `HSA_*` secrets (root:group, 0640)        |
| `/opt/headscale-admin/data/`           | `headscale.work_dir`: headscale config, sqlite DB, keys, socket |
| `/var/log/headscale-admin/`            | logs (rotated via logrotate, copytruncate)|

All paths are overridable (`headscale_admin_home`, `headscale_admin_bin_dir`,
`headscale_admin_work_dir`, `headscale_admin_log_dir`, …).

## 1. Installing the binary

Select the source with `headscale_admin_install_source`:

```yaml
# GitHub release (default)
headscale_admin_install_source: github
headscale_admin_version: latest          # or a tag like v0.3.0
headscale_admin_repo: YouSysAdmin/headscale-admin
# headscale_admin_github_token: "ghp_..."   # optional

# Direct URL
headscale_admin_install_source: url
headscale_admin_download_url: "https://example.com/headscale-admin-{{ headscale_admin_os }}-{{ headscale_admin_arch }}"

# S3 (requires the amazon.aws collection: ansible-galaxy collection install amazon.aws)
headscale_admin_install_source: s3
headscale_admin_s3_bucket: my-bucket
headscale_admin_s3_object: headscale-admin/headscale-admin-linux-amd64
headscale_admin_s3_region: eu-central-1
# headscale_admin_s3_endpoint_url: "https://minio.example.com"   # MinIO etc.
```

The S3 download needs only `s3:GetObject` on the object - the role sets
`ignore_nonexistent_bucket` so it skips the `s3:ListBucket`/HeadBucket pre-flight
(otherwise a least-privilege role fails with *"Permission denied accessing bucket
… (during bucket_check)"* even though `aws s3 cp` works). A minimal policy:

```json
{ "Effect": "Allow", "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/headscale-admin/*" }
```

The GitHub asset name is resolved as
`headscale-admin-{{ os }}-{{ arch }}` (only `linux`/`darwin` × `amd64`/`arm64`
are published). A `.tar.gz`/`.zip` asset is also accepted and unpacked.

**Integrity:** set `headscale_admin_sha256` for an explicit checksum (wins over
everything). For the GitHub source the role also parses a `checksums.sha256`
asset if present (`headscale_admin_checksums_asset`). Set
`headscale_admin_checksum_required: true` to fail when no checksum can be
determined.

## 2. headscale-admin config

Non-secret settings go to `config.yaml` (`server` / `headscale` / `auth` /
`logging`); **secrets go to the env file** as `HSA_*` overrides and are never
written to `config.yaml`.

```yaml
headscale_admin_server_addr: ":443"
headscale_admin_public_url: "https://hs.example.com"

# TLS at the front (Let's Encrypt)
headscale_admin_tls_mode: acme
headscale_admin_tls_acme_email: ops@example.com
headscale_admin_tls_acme_hosts: ["hs.example.com"]

# Auth - the session-cookie HMAC secret (kept out of config.yaml). Login is
# per-user: create the first admin after install with
# `headscale-admin admin create` (email+password), or use OIDC below.
headscale_admin_session_secret: "{{ vault_hsa_session_secret }}"   # >= 32 chars

# …or OIDC login for the admin UI:
headscale_admin_oidc_enabled: true
headscale_admin_oidc_issuer: "https://accounts.example.com"
headscale_admin_oidc_client_id: "headscale-admin"
headscale_admin_oidc_client_secret: "{{ vault_hsa_oidc_secret }}"  # -> env file
headscale_admin_oidc_redirect_url: "https://hs.example.com/admin/api/auth/oidc/callback"
# RBAC: map IdP groups (and bootstrap admins) to roles. admin_emails also pins
# those users to admin, so they don't need `admin create` when OIDC is on.
headscale_admin_oidc_admin_emails: ["ops@example.com"]
headscale_admin_oidc_admin_groups: ["hs-admins"]
headscale_admin_oidc_operator_groups: ["hs-operators"]
headscale_admin_oidc_groups_claim: "groups"
```

To IP-restrict the admin UI but keep device onboarding public, set
`headscale_admin_ui_allow_ips` **and** `headscale_admin_public_signup: true` -
the signup page (`/admin/signup`) and its static assets bypass the allowlist
while the dashboard, login, and data APIs stay gated.

The bare public root `/` is configurable via `headscale_admin_root_*` (leave
`headscale_admin_root_mode` empty to keep the default `302` → signup):

```yaml
# Redirect "/" to the sign-in page - or any URL, e.g. an internal wiki:
headscale_admin_root_mode: redirect
headscale_admin_root_target: signin   # signup | signin | dashboard | /path | https://…
# headscale_admin_root_status: 302    # 301/302/303/307/308 (default 302)

# …or serve an inline page instead of redirecting:
# headscale_admin_root_mode: page
# headscale_admin_root_status: 404
# headscale_admin_root_body: "Nothing to see here."
# headscale_admin_root_content_type: "text/plain; charset=utf-8"  # or text/html
```

Logs are written to `headscale_admin_log_output`
(`/var/log/headscale-admin/headscale-admin.log` by default; set to `stdout` to
log to the journal instead, which also skips the logrotate config).

> Store secrets with Ansible Vault. When auth is enabled you must provide a
> session secret and at least one login method (OIDC or a static token), or set
> `headscale_admin_auth_disabled: true` (not for production).

## 3. Pre-configuring the managed headscale (optional)

By default headscale-admin generates `headscale.yaml` on first run and owns it
(the Settings UI edits it in place). To pre-seed it from Ansible instead:

```yaml
headscale_preconfigure: true
# false: write only if headscale.yaml does not exist yet (preserves UI edits)
# true:  Ansible owns the file and overwrites on every run
headscale_preconfigure_force: false

headscale_dns_base_domain: "ts.example.com"
headscale_dns_nameservers: ["1.1.1.1", "9.9.9.9"]
headscale_policy_mode: file        # or keep "database" for the visual editor
```

This writes `/opt/headscale-admin/data/headscale.yaml`. The loopback listeners,
`grpc_allow_insecure`, trusted proxies, socket and on-disk paths are fixed to
what headscale-admin expects - change the DNS / DERP / policy / prefix knobs,
not the listeners. There is intentionally **no TLS section** here: TLS is
terminated by headscale-admin at the front.

DERP maps can be rendered from `headscale_derp_maps`:

```yaml
headscale_derp_maps:
  - filename: derp-custom.yaml
    regions:
      900:
        regionid: 900
        regioncode: custom
        regionname: My Region
        nodes:
          - name: "900a"
            regionid: 900
            hostname: "derp.example.com"
            ipv4: "198.51.100.1"
            ipv6: "2001:db8::1"
```

headscale's *own* OIDC (so tailscale clients authenticate via your IdP - this
is independent from the admin-UI login above) is available via the
`headscale_oidc_*` variables when `headscale_oidc_enabled: true`. By default the
client secret you put in `headscale_oidc_client_secret` is written to
`headscale_oidc_client_secret_path` (a `0600` file under `work_dir`) and
referenced from `headscale.yaml` via `client_secret_path`, keeping it out of the
config file. Set `headscale_oidc_client_secret_path: ""` to inline it instead.

To pin headscale's identity keys instead of letting it generate them on first
start, provide the key contents - the role writes them to `0600` files that
`headscale.yaml` references:

```yaml
headscale_noise_private_key: "{{ vault_hs_noise_key }}"
headscale_derp_private_key: "{{ vault_hs_derp_key }}"   # only with DERP enabled
```

> These key/secret files are written as part of the pre-configure flow, so they
> require `headscale_preconfigure: true`. Store the values with Ansible Vault.

## 4. External-user sync (headscale-pf) (optional)

Enable the **User sync** page: headscale-admin downloads the pinned
[headscale-pf](https://github.com/YouSysAdmin/headscale-pf) release (into
`bin/`) and fills the policy's `group:` members from an identity source.
Non-secret settings go to `config.yaml`; the token / LDAP bind password are
written to the env file as `HSA_HEADSCALE_PF_*` (keep them in Vault).

```yaml
# Jumpcloud (its hosted API needs no endpoint):
headscale_admin_pf_enabled: true
headscale_admin_pf_source: "jc"
headscale_admin_pf_token: "{{ vault_pf_token }}"     # -> env file

# Authentik / Keycloak / LDAP need an endpoint:
# headscale_admin_pf_source: "ldap"
# headscale_admin_pf_endpoint: "ldaps://dir.example.com:636"
# headscale_admin_pf_ldap_base_dn: "dc=example,dc=com"
# headscale_admin_pf_ldap_bind_dn: "cn=readonly,dc=example,dc=com"
# headscale_admin_pf_ldap_bind_password: "{{ vault_pf_ldap_password }}"  # -> env file
# headscale_admin_pf_ldap_default_email_domain: "example.com"
# headscale_admin_pf_keycloak_realm: "my-realm"      # kk source
```

A sync rewrites the policy's group members from this one source, so configure
exactly one. Pin/override the tool version with `headscale_admin_pf_version`, or
supply your own binary via `headscale_admin_pf_binary` (air-gapped; disables UI
updates).

## Example play

```yaml
- hosts: headscale
  become: true
  roles:
    - role: headscale-admin
      vars:
        headscale_admin_public_url: "https://hs.example.com"
        headscale_admin_server_addr: ":443"
        headscale_admin_tls_mode: acme
        headscale_admin_tls_acme_email: ops@example.com
        headscale_admin_tls_acme_hosts: ["hs.example.com"]
        headscale_admin_session_secret: "{{ vault_hsa_session_secret }}"
        headscale_preconfigure: true
        headscale_dns_base_domain: "ts.example.com"
```

## Notes

- First start needs outbound network (it downloads the pinned headscale
  release into `bin/`).
- With auth enabled, headscale-admin refuses to start until an admin exists.
  After the first install, create one on the host:
  `sudo -u headscale-admin headscale-admin admin create --config <home>/config.yaml`
  (prompts for email+password). When OIDC is enabled, set
  `headscale_admin_oidc_admin_emails` instead - no password needed.
- The systemd unit uses `KillMode=control-group`, so headscale (the child
  process) stops together with headscale-admin.
- The managed headscale version is pinned by headscale-admin; leave
  `headscale_admin_headscale_version` empty unless you understand the gRPC
  proto coupling.
