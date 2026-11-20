# Changelog

## 1.0.0 (2026-11-18)

First public release in the OpenNebula GitHub organization.

- 12 OpenNebula hooks send evidence for VMs, NICs, disks, images, Virtual Networks, users and datastores to the
  Confirmate Evidence Store.
- Evidence covers CIS 4.3, 4.4, 4.5, 8.3, 9.2 and 9.3, plus backups and log retention, and in part CIS 8.5 and 8.6.
  See [docs/mapping.md](docs/mapping.md).
- OAuth2 client credentials (for example Keycloak), an optional extra CA file, and copies to additional Targets of
  Evaluation.
- Passwords, tokens and 2FA secrets are removed from the raw XML before it is sent.
