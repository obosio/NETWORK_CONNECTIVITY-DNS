# WEB proxy DNS-01 grant

## Change record

- Date: October 23, 2026
- Zone and view: External `intelisis.com.ar`
- Primary: CT `25100` on `pve-upr-3-luj`
- Secondary: CT `26100` on `pve-upr-3-cen`
- Approval: operator-approved exact-owner TXT-only challenge grant and renewal
- Protected pre-change configuration:
  `/root/network-manager-web-proxy-acme-prechange-20261023T2125Z/named.conf.local`

## Scope

The dedicated TSIG identity is `web-proxy-acme`. Its key file is stored only
on the primary at `/etc/bind/web-proxy-acme.key`, owned by `root:bind` with
mode `0640`. The secret is intentionally not recorded here.

The primary configuration:

- Includes the dedicated key.
- Excludes the identity from the Internal view so it selects External.
- Grants only TXT updates at
  `_acme-challenge.web-proxy.intelisis.com.ar.` in the External
  `intelisis.com.ar` zone.

The change did not publish an A/CNAME record, alter the router's gateway or
DNS settings, or configure any tenant. The existing `web-acme` identity
remains limited to its original challenge owner and was not reused.

## Validation and resulting state

- Read-only pre-change checks found the primary and secondary zones active
  with matching External serial `2026102209`.
- A protected copy of `named.conf.local` was created before modification.
  Its SHA-256 matched the live pre-change file:
  `a2e67a239835b11c9adf23f2d8534a4965eecc4325cfd7972184bc2589f7d7c7`.
- The reviewed diff contained only the key include, Internal-view exclusion,
  and exact External TXT grant.
- `named-checkconf -z` validated all configured zones.
- `rndc reconfig` applied the change without restarting BIND.
- A temporary TXT update at the exact owner succeeded, was queried through
  the signed External view, and was removed. The challenge owner returned no
  TXT answer after cleanup.
- A TXT update at a sibling challenge owner was refused.
- The primary and secondary External zone serials now match at `2026102211`.
- The attempt to test a non-TXT RR type with `nsupdate` was rejected locally
  by its check-names validation before an update packet was sent; therefore
  runtime denial of another RR type was not independently demonstrated. The
  live BIND policy is explicitly restricted to the TXT type, but a separate
  protocol-level non-TXT denial test remains a validation item.

## Credential handoff and issuance

The key remains on the DNS primary. It has not been transferred to the WEB
service or another application, and no certificate has been issued. Before
staging or production issuance, transfer the credential to the eventual
service through the approved protected handoff procedure, with the recipient
path and ownership established first. Do not copy the key into the source
tree, browser, database, image, or logs. Follow the
[RFC2136 credential procedure](../../NETWORK_MANAGER/doc/RFC2136_CREDENTIAL_PROCEDURE.md)
for secret custody and staging issuance.

No public A/CNAME record or persistent challenge TXT record is present as a
result of this grant operation. Certificate Transparency will disclose the
hostname upon public-CA issuance; no certificate has been requested yet.
