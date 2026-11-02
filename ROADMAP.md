# Roadmap

The goal is to make Rauthy plus rauthy-ldap a drop-in replacement for LLDAP, with Jellyfin and
Authelia as the first LDAP clients. Each milestone ends with something usable.

## 0. Scaffold

- [ ] Cargo workspace on edition 2024: binary crate `rauthy-ldap`, built on
      [`ldap3_proto`](https://crates.io/crates/ldap3_proto) and tokio
- [ ] Start from the `ldap3_proto` tokio example server
- [ ] CI: `fmt`, `clippy -D warnings`, tests, `cargo deny` (licenses, advisories)
- [ ] Multi-stage Dockerfile: static binary on distroless, non-root
- [ ] Configuration from environment variables, validated at startup

## 1. Minimum viable gateway: Jellyfin

- [ ] Rauthy PAM API client: `getent` (`Users`, `Username`, `Groups`, `Groupname`) and `login`
- [ ] Service-account bind, checked locally
- [ ] User bind → `POST /auth/v1/pam/login`, with password or remote password
- [ ] Search on `ou=people` and `ou=groups`, LLDAP-compatible layout and attributes
- [ ] Filter evaluation: `and`, `or`, `not`, equality, presence, substring
- [ ] In-memory cache for `getent` results with a TTL
- [ ] All write operations and SASL refused with `unwillingToPerform`
- [ ] Integration test: Rauthy container + gateway, replaying Jellyfin LDAP-Auth's sequence:
      service bind → user search → user bind → admin-group search
- [ ] Manual test against a real Jellyfin with the LDAP-Auth plugin

## 2. Authelia compatibility

- [ ] Works with Authelia `implementation: lldap` defaults:
  - users filter `(&(|(uid={input})(mail={input}))(objectClass=person))`
  - groups filter `(&(member={dn})(objectClass=groupOfUniqueNames))`
  - attributes `uid`, `cn`, `mail`, `memberOf`
- [ ] Group search by member DN, `memberOf` on users
- [ ] Documented limit: Authelia password reset must be disabled, because the directory is
      read-only
- [ ] Integration test with Authelia's LDAP backend

## 3. Hardening

- [ ] LDAPS and StartTLS; refuse plaintext binds unless explicitly allowed
- [ ] Forward the client's IP to Rauthy, so its brute-force and credential-stuffing protection
      sees the real client rather than the gateway
- [ ] Connection, request-size and per-IP limits
- [ ] Constant-time comparison for the service-account secret; zeroize passwords
- [ ] Structured logs without credentials; `/health` and Prometheus metrics
- [ ] Fuzz the LDAP request path
- [ ] Threat model in `SECURITY.md`

## 4. Compatibility matrix

- [ ] Test more clients from LLDAP's
      [`example_configs`](https://github.com/lldap/lldap/tree/main/example_configs), starting
      with Gitea, Grafana, Home Assistant, Nextcloud and Proxmox
- [ ] Per-application setup notes in `docs/`
- [ ] Per-client scoping: different bind accounts see only the groups they are configured for

## 5. Migration tool: LLDAP → Rauthy

A separate binary `rauthy-ldap-import`, as suggested in rauthy#1849, so the gateway itself needs
no LDAP client code.

- [ ] Read users and groups from LLDAP (LDAP or GraphQL)
- [ ] Create the matching Rauthy users, PAM users and PAM groups through the Rauthy API, with
      group memberships
- [ ] Dry-run by default; idempotent re-runs
- [ ] Passwords are not migrated: LLDAP stores OPAQUE registrations, not hashes.
      Users set a password or passkey in Rauthy.

## 6. Migration guide: LLDAP → Rauthy

- [ ] Document running rauthy-ldap next to LLDAP and testing a client against it
- [ ] Document moving clients (Jellyfin, then Authelia) from LLDAP to rauthy-ldap
- [ ] Document retiring LLDAP

## Open questions

- Remote passwords expire after two minutes by default. That works for applications that bind
  once and keep a session, like Jellyfin, but not for applications that re-bind on every
  request. Does Rauthy want longer-lived, per-application passwords for that case?
- The PAM API returns no given or family name, so `cn` and `displayName` use the username.
  Should `getent` return them?
- Should the project live under `sebadob/` once it is stable? Discuss in rauthy#1849.
