# NETWORK_MANAGER router DNS-01 grants

## Change record

- Date: October 22, 2026
- Zone and view: External `intelisis.com.ar`
- Primary: CT `25100` (`ns1.intelisis.com.ar`) on `pve-upr-3-luj`
- Secondary: CT `26100` (`ns2.intelisis.com.ar`) on `pve-upr-3-cen`
- Approval: the owner approved four independent identities, each limited to
  TXT updates at one exact ACME challenge owner
- Protected pre-change configuration: primary CT `25100`,
  `/root/network-manager-router-acme-prechange-20261022/`

## Installed identities and policy

Each private key is stored only on the primary in `/etc/bind`, owned
`root:bind`, mode `0640`. Secret values are not included in this record.

| Router | TSIG identity | Primary key file | Sole permitted update |
| --- | --- | --- | --- |
| TIC | `network-manager-router-tic-acme` | `/etc/bind/network-manager-router-tic-acme.key` | `_acme-challenge.tic.intelisis.com.ar.` TXT |
| Lujan | `network-manager-router-lujan-acme` | `/etc/bind/network-manager-router-lujan-acme.key` | `_acme-challenge.lujan.intelisis.com.ar.` TXT |
| Centro | `network-manager-router-centro-acme` | `/etc/bind/network-manager-router-centro-acme.key` | `_acme-challenge.centro.intelisis.com.ar.` TXT |
| Cordoba | `network-manager-router-cordoba-acme` | `/etc/bind/network-manager-router-cordoba-acme.key` | `_acme-challenge.cordoba.intelisis.com.ar.` TXT |

The four identities are excluded from the Internal view. The External
`update-policy` contains only the corresponding exact owner/type grant for
each identity. Existing identities, policies, records, and dynamic zone files
were preserved.

## Validation and resulting state

- `named-checkconf -z` passed before activation.
- `rndc reconfig` applied the configuration without restarting BIND.
- For every identity, a temporary TXT update at its own owner succeeded and
  was removed.
- For every identity, an unknown non-TXT record type at its own owner and a
  TXT update at another router's challenge owner were denied.
- All four challenge owners returned authoritative NXDOMAIN after cleanup.
- The primary and secondary External `intelisis.com.ar` zones report matching
  serial `2026101740`.
- `rndc status` confirmed that the primary remained up and running.

Following separate owner approval, the four matching RFC2136 credentials were
transferred to NETWORK_MANAGER CT `27030` and installed as
`network-manager`-owned files, mode `0600`. Let's Encrypt staging issuance
succeeded for all four exact hostnames; each temporary challenge was removed
and all four owners returned authoritative NXDOMAIN afterward. The staging
certificates remain in the application's isolated state directory and were
not installed on routers. Primary and secondary External zone serials
advanced together to `2026101748`. No production issuance, router
configuration, A/CNAME record, NAT, or firewall policy was changed. The ACME
account was registered without an email address; expiry notifications are
therefore not available.

For the application-side status and credential handoff boundary, see the
[NETWORK_MANAGER RFC2136 procedure](../../NETWORK_MANAGER/doc/RFC2136_CREDENTIAL_PROCEDURE.md).
The cluster operation record is in the
[PVE DNS primary/secondary log](../../../INFRASTRUCTURE/PVE/doc/DNS_PRIMARY_SECONDARY.md).
