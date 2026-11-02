# rauthy-ldap

A small, read-only LDAP gateway for [Rauthy](https://github.com/sebadob/rauthy), written in Rust.

> **Status: design phase.** Nothing is implemented yet. This README describes what the gateway
> will do; [ROADMAP.md](ROADMAP.md) tracks how it gets there.

## Why

Some applications can only check a username and password against an LDAP directory. Media
servers are the usual example: Jellyfin's TV, mobile and desktop apps send a username and
password to the Jellyfin server, which checks them with its LDAP plugin. Today that means
running a second user directory, such as LLDAP, next to your identity provider, and the two
drift apart.

rauthy-ldap lets those applications use Rauthy as their directory instead, so Rauthy stays the
single source of truth for users and groups. It is the missing piece for replacing LLDAP with
Rauthy.

Background: [sebadob/rauthy#1849](https://github.com/sebadob/rauthy/discussions/1849).

## How it works

rauthy-ldap is a separate process. It speaks LDAP to applications and calls Rauthy's existing
PAM API over HTTPS. It is a third client of that API, next to the PAM and NSS modules in
[rauthy-pam-nss](https://github.com/sebadob/rauthy-pam-nss), so Rauthy needs no code changes and
no LDAP code in its core.

```
app (e.g. Jellyfin) ──LDAPS :636──▶ rauthy-ldap ──HTTPS /auth/v1/pam/*──▶ Rauthy
                                    (PAM host id + secret)
```

| LDAP operation | Rauthy PAM API |
|---|---|
| Service-account bind | Checked locally against the configured bind DN and secret |
| User bind | `POST /auth/v1/pam/login` with the password or a PAM remote password |
| Search users and groups | `POST /auth/v1/pam/getent` (`Users`, `Username`, `Groups`, `Groupname`), cached in memory |
| Access control | The user must be in a PAM group that is allowed on the gateway's PAM host |

The gateway registers in Rauthy as an ordinary **PAM host**. Rauthy's rules apply unchanged:
- Who may log in is decided by the PAM groups linked to that host.
- Accounts with a passkey or MFA can't bind with their main password. Instead, they generate a
  **PAM remote password** in the Rauthy account dashboard. It is short-lived, two minutes by
  default (`remote_password_ttl`).
- Brute-force and credential-stuffing protection stays in Rauthy.

Short-lived remote passwords suit applications that bind once and then keep their own session,
like Jellyfin. They do not suit applications that re-bind with the stored password on every
request.

## What it supports

**Supported (read-only LDAPv3 subset)**
- Simple bind (service account and users), unbind
- Search with base, one-level and subtree scope
- Filters: `and`, `or`, `not`, equality, presence and substring
- RootDSE and the Who Am I extended operation (RFC 4532)
- LDAPS, or StartTLS on a plain listener

**Refused with `unwillingToPerform`**
- Add, modify, delete, modify DN, compare
- SASL binds
- Password changes. Passwords are managed in Rauthy.

## Directory layout

The layout copies LLDAP's, so the presets that applications already ship for LLDAP work
unchanged. Authelia's `implementation: lldap` and Jellyfin's LLDAP example config are the first
targets.

```
dc=example,dc=com
├── ou=people
│   └── uid=<pam username>
│        objectClass: person, inetOrgPerson, posixAccount
│        uid, cn, displayName, mail, uidNumber, gidNumber, homeDirectory, loginShell, memberOf
└── ou=groups
    └── cn=<pam group name>
         objectClass: groupOfUniqueNames, groupOfNames, posixGroup
         cn, gidNumber, member, uniqueMember
```

`cn` and `displayName` fall back to the PAM username, because the PAM API does not return given
or family names.

## Configuration (planned)

| Variable | Description |
|---|---|
| `RAUTHY_URL` | Base URL of Rauthy, e.g. `https://auth.example.com` |
| `RAUTHY_PAM_HOST_ID` | ID of the PAM host created for the gateway |
| `RAUTHY_PAM_HOST_SECRET` | Secret of that PAM host |
| `LDAP_BASE_DN` | e.g. `dc=example,dc=com` |
| `LDAP_BIND_DN` / `LDAP_BIND_PASSWORD` | Service account that applications use for searches |
| `LDAP_LISTEN` | Listen address, default `0.0.0.0:3636` |
| `LDAP_TLS_CERT` / `LDAP_TLS_KEY` | Certificate for LDAPS |
| `CACHE_TTL` | How long `getent` results are cached |

## Setup in Rauthy

1. Enable PAM in Rauthy.
2. Create a PAM host, e.g. `ldap-gateway`, and copy its ID and secret.
3. Create PAM groups, e.g. `jellyfin`, link them to the host and add users.
4. Users with a passkey or MFA generate a remote password in their account dashboard when an
   application asks them to log in.

## Deployment

rauthy-ldap is a static binary in a distroless, non-root container. Run it as its own container
or Kubernetes Deployment next to Rauthy, not inside Rauthy's container or pod.
- Expose only the LDAP port.
- Allow only the gateway to reach Rauthy, and only the LDAP clients to reach the gateway.

## Migrating from LLDAP

See [ROADMAP.md](ROADMAP.md): a separate import tool copies users and groups from LLDAP into
Rauthy. Passwords can't be copied, because LLDAP stores them as OPAQUE registrations, so users
set a new password or a passkey in Rauthy.

## License

[MPL-2.0](LICENSE)
