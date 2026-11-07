# DNS_ENVIRONMENT

Updated: Oct 23, 2026

**DNS infrastructure, authoritative zone reconciliation, and operational audit platform** for the primary and secondary BIND DNS services, zone replication, and split-view publication across all network and cluster boundaries.

Core system: Authoritative and recursive DNS infrastructure and zone reconciliation system
Domain: Network naming services, DNS infrastructure, and IP reconciliation
Core runtime: BIND 9 on Debian 13 guests (CT 25100 and CT 26100) with Python inspection and reconciliation tooling
Operational placement: Proxmox VE cluster hosts (Lujan CT 25100 and Centro CT 26100), core edge routers, and public DNS interfaces
Primary transport: DNS protocol (UDP/TCP 53), TSIG AXFR/IXFR, and SSH
Supporting components: Zone reconciliation models, validation tooling, inspection probes, migration checklists, and operational documentation

This project exists to deliver managed authoritative and recursive DNS infrastructure and zone reconciliation; inspection probes, inventory data, and migration helpers are supporting mechanisms and do not define the project identity.

## Scope

On November 7, 2026, the existing Network Manager TIC CT `27030` migrated to
`netmgr-tic.intelisis.com.ar` (Internal A `10.8.0.61`, TTL `300`). Its HTTPS
and SSH use internal high ports `20302` and `20303`; the new name is absent
from the External view. The generic `netmgr.intelisis.com.ar` was removed
from both views and authoritative NXDOMAIN verified on both servers.
The primary/secondary serials match: Internal `2026110704`, External
`2026110701`. Only the exact External TXT grant for
`_acme-challenge.netmgr-tic.intelisis.com.ar.` remains for the existing
`network-manager-web-acme` identity. Production TLS issuance and renewal
dry run passed; see the [application deployment record](../NETWORK_MANAGER/doc/LXC_DEPLOYMENT.md).
Use site-qualified `netmgr-<site>` identities for instances, not a shared
generic hostname. No other site's instance or DNS name was provisioned.

- Maintain primary and secondary authoritative BIND 9 services across sites (`ns1.intelisis.com.ar` on CT 25100, `ns2.intelisis.com.ar` on CT 26100).
- Manage split-view policy (Internal view for RFC1918 and internal networks; External view for public authoritative domains).
- Reconcile authoritative forward and reverse records against live edge router address and destination-NAT state.
- Inspect and monitor live zone serial synchronization, TSIG authenticated replication, and query consistency across views.
- Provide a clean migration path and approval boundaries for public IP reorganization and DNS cutovers.

On October 22, 2026, four exact-scope External RFC2136 grants were provisioned
for NETWORK_MANAGER router certificate challenges. The identities are limited
to TXT updates at their respective `_acme-challenge` owners. Subsequent
staging issuance for all four profiles cleaned up the temporary challenge
records; the primary and secondary External zone serials match at
`2026101748`. See the
[router grant operation record](doc/NETWORK_MANAGER_ROUTER_DNS_GRANTS_2026-10-22.md).

On October 23, 2026, an operator-approved External grant was provisioned for
the WEB proxy certificate challenge. Its dedicated identity is limited to TXT
at `_acme-challenge.web-proxy.intelisis.com.ar.`. The temporary exact-owner
TXT test was removed, sibling-owner TXT was refused, and primary/secondary
serials match at `2026102211`. The key remains on the DNS primary; no WEB
credential handoff or certificate issuance has occurred. See the
[WEB proxy grant operation record](doc/WEB_PROXY_DNS_GRANT_2026-10-23.md).

The Database DNS/TLS cutover and Mantis TIC CNAME cutover are live and
reconciled with the current service state. Database internal-client acceptance
passed; external route reachability remains waived and unverified. Mantis
resolves to TIC in both views, with its client forwarding active on TIC and no
Mantis forwarding found on Lujan. October 16 checks from this workstation
reached its HTTPS login redirect and SSH banner; a subsequent authenticated
web login reached the dashboard with TLS verification enabled. No issue or
customer records were read or changed. The operator confirms ingress access
control is enforced by the CORE routers' RouterOS `/ip firewall filter`
rules; there is no separate external firewall appliance. The completed
and pending work is documented in
[`config/tic_workload_dns_cutover_plan.yaml`](config/tic_workload_dns_cutover_plan.yaml).
Database service lifecycle and PostgreSQL migration planning are owned by
[DB_ENVIRONMENT](../../SERVICES/DB/README.md); this project retains ownership
of the DNS records, DNS/TLS cutover state, and authoritative verification.
The shared Aero and Intelisis apexes now match their approved split-view
targets and were verified on both authorities. Lujan TCP `80,443` forwarding
targets TIC Gestion CT `27020`; unassigned/disabled customer-port mappings
remain unchanged pending owner review.
The approved October 14 cleanup retired the legacy `database` PTR and the
obsolete `services`, `fileserver`, `owncloud`, and `wiki` Aero records.
`mantis.aero.com.ar` and `gestion.aero.com.ar` remain absent. Authoritative
NXDOMAIN was verified for these names in both views on both DNS authorities.
On October 16, the deprecated `groupware.aero.com.ar` Internal A record was
removed and authoritative NXDOMAIN verified on both Internal authorities at
SOA `2026101601`. Public resolvers already returned NXDOMAIN, so the External
zone was left unchanged. The production Mailcow identity
`mail.aero.com.ar` still resolves to the approved mail service in both views;
the guest at `192.168.25.40` was not changed.
The currently published production database identity remains
`db1.intelisis.com.ar`; on October 17, the operator selected future names
`db-tic.intelisis.com.ar` for existing TIC VM 27025 and
`db-cor.intelisis.com.ar` for existing Córdoba VM 24025. These names have not
been published or applied to guest hostnames. DNS, certificate, database-role,
and application-client transitions are pending a sequenced plan. The old
`138.121.101.4` assignment and its dynamic `/32` route have now been removed
from all four CORE routers; no old `5432` NAT mapping remains. PMS records were
removed from both DNS views and its External PTR was deleted. PBX now has only
its Internal A record (`192.168.26.6`); no External A or CORE destination NAT
remains. Polodesalud's existing apex and PPPoE route were confirmed correct by
the operator and left unchanged. The detailed actions and verified zone
serials are recorded in
[`doc/EXTERNAL_DNS_SERVICE_RECONCILIATION.md`](doc/EXTERNAL_DNS_SERVICE_RECONCILIATION.md).

On October 15, the External A record for `iot.intelisis.com.ar` was published
at `138.121.100.6`, and `ceis.polodesalud.com.ar` was published at
`138.121.100.8`. IoT firmware HTTP and HTTPS/MQTT TLS endpoints are now
verified. On October 16, the operator approved a forced production renewal.
The dedicated IoT-local Home Assistant Certbot app renewed the certificate
through DNS-01/RFC2136, deployed it to Home Assistant OS `/ssl`, and restarted
Home Assistant and Mosquitto. The new certificate fingerprint matches the
deployed files and the certificates served by public HTTPS and MQTT TLS;
hostname and chain checks passed, HTTPS returned 200, and both public DNS
resolvers show no remaining challenge TXT record. The External
`intelisis.com.ar` SOA serial is `2026101522` on both authorities. The staging
dry-run issue against the production lineage did not recur during the real
production renewal. The dedicated IoT administrative SSH path on TCP `20073` is also verified;
the retired public TCP `8922` SSH forward remains closed while MQTT `8883`
stays enabled. Keep the HTTP-01 add-on stopped. CEIS remains operational and
its public destination-NAT rule is limited to TCP `80`/`443`; unused TCP
`12321`/`12323` and UDP `12321` are not published.
Internal IoT DNS remains `192.168.25.7`; its External PTR now matches `.100.6`,
and the former `.101.3` PTR was removed.

On October 16, GPS administration was moved to the high-port path
`138.121.100.2:20083` -> `192.168.25.8:22`; the dedicated environment key
authenticated successfully. GPS remains on its own public IP and client ports.
Ollama is a test service published only in Internal DNS as
`ollama.intelisis.com.ar` -> `192.168.27.10`; public DNS remains NXDOMAIN and
no public Ollama address, route, or NAT is configured. Prestoprenda remains
customer-DNS-owned; the operator confirmed its external path is already
verified, so no Internal record or local DNS cutover is pending.

On October 16, the operator directed retirement of
`projects.intelisis.com.ar` from all managed DNS views. Its Internal and
External A records and External PTR were removed and the updated zones
replicated to both authorities. The Projects guest remains deprecated and
powered off; its router assignment and broad destination NAT were not changed
by this DNS-scoped action. Do not republish the name or treat the address as
available.

S3 DNS analysis and any cutover are deferred until the currently ongoing load
balancer deployment is complete. That deployment is external to this project’s
scope; retain the current S3 DNS state until a later, separately scoped review.

The read-only reconciliation of the workload high-port scheme, DNS answers,
live NAT rules, and six CORE router management services is documented in
[`doc/HIGH_PORTS_RECONCILIATION_2026-10-16.md`](doc/HIGH_PORTS_RECONCILIATION_2026-10-16.md).
It distinguishes active mappings from registry plans and records the
outstanding DNS, allocation-policy, and router-service discrepancies.

## Architecture & Topology

| Role | Container | Site | IP Address | Host Node |
| --- | --- | --- | --- | --- |
| Primary (Master) | CT 25100 (`ns1.intelisis.com.ar`) | Lujan | `192.168.25.100` / `138.121.100.1` | `pve-upr-3-luj` |
| Secondary (Replica) | CT 26100 (`ns2.intelisis.com.ar`) | Centro | `192.168.26.100` / `138.121.102.1` | `pve-upr-3-cen` |

### Authoritative Zones

Ten zones are replicated across two distinct views:
- **Internal View**: `intelisis.com.ar`, `aero.com.ar`, `polodesalud.com.ar`
- **External View**: The same three forward zones plus reverse zones `100.121.138.in-addr.arpa`, `101.121.138.in-addr.arpa`, `102.121.138.in-addr.arpa`, `103.121.138.in-addr.arpa`.

`pro3d.com.ar` was retired from both BIND views on October 13, 2026. The domain,
its parent delegation, and any future DNS or mail service are now managed by
its owner and are outside this project's scope. No further action on Pro3D is
pending from DNS_ENVIRONMENT.

## Tooling & Inspection

`config/external_dns_reconciliation.yaml` is the canonical DNS-owned
reconciliation ledger. MIKROTIK-MANAGER validates its router-owned
destination-NAT registry independently; it does not maintain a second copy of
this DNS reconciliation data.

A Python CLI tool (`dns-environment`) provides deterministic validation of the reconciliation ledger and live DNS inspection:

```bash
# Validate reconciliation data
.venv/bin/dns-environment validate --reconciliation config/external_dns_reconciliation.yaml

# Check SOA answers at configured DNS server addresses
.venv/bin/dns-environment inspect

# Run test suite
.venv/bin/pytest
```

The inspection command queries SOA records without recursion and only treats
responses carrying the authoritative-answer flag as valid. Use `--reverse` to
include reverse zones, checked against External-view servers only. When run
from an RFC1918 client, BIND may intentionally select its Internal view even
for a public server address; run reverse inspection from an external vantage
to verify public-view behavior. This unsigned command is source/view dependent;
it does not enumerate both split views. Use view-specific `rndc zonestatus`
and External-TSIG queries via the PVE access procedure below when reconciling
both views.

### Accessing DNS guests for operational verification

Run guest-level checks through the Proxmox VE host that currently runs the
target LXC, using the PVE project's inventory and credential procedure. After
connecting to the trusted PVE host, use read-only commands through `pct exec`,
for example:

```bash
pct exec 25100 -- rndc status
pct exec 25100 -- rndc zonestatus 100.121.138.in-addr.arpa IN External
```

Use CT `25100` on `pve-upr-3-luj` for the primary and CT `26100` on
`pve-upr-3-cen` for the secondary, confirming current placement before access.
Do not connect directly to an LXC from the development workstation, bypass SSH
host-key verification, or expose credentials or TSIG material in command
output. For External-view checks from an RFC1918 source, use the scoped
External TSIG identity without printing its secret; unsigned requests may
select the Internal view according to BIND's client ACLs.

## APPEAR Internal record (November 5, 2026)

The operator initially requested publishing `appear.intelisis.com.ar` in the system BIND
DNS. Its initial Internal-only A record was `192.168.15.105`, TTL 300. Primary CT 25100
on `pve-upr-3-luj` loaded serial `2026110503` at `04:49:13 UTC`; secondary
CT 26100 on `pve-upr-3-cen` loaded the same serial automatically. Both servers
returned authoritative answers, and gateway resolver `192.168.15.1` returned
the same address. No External record or PTR was created.

The first reload failed because the existing static Internal zone file was
`root:root`, mode `0600`, unreadable by BIND. Its on-disk contents were rolled
back with a forward-only serial before retrying. The successful candidate
uses `root:bind`, mode `0640`. Zone and full configuration validation passed;
the External zone and `named.conf.local` hashes were unchanged. Protected
pre-change backups remain on the primary under
`/root/appear-dns-prechange-20261105T044814Z` and
`/root/appear-dns-prechange-20261105T044913Z`. No service restart was performed.
Secondary `zonestatus` and authoritative responses verify replication;
the targeted syslog check returned no transfer lines.

The operator corrected that direct-to-device publication to the existing CORE
high-port NAT scheme, then approved HTTP after native HTTPS remained unverified.
The installed endpoint is **http://appear.intelisis.com.ar:21051**:

| View | A record | CORE translation |
| --- | --- | --- |
| Internal | `10.8.0.51` | TCP 21051 to `192.168.15.105:80` |
| External | `45.238.8.15` | TCP 21051 to `192.168.15.105:80` |

Both use TTL 300. At `05:10:40 UTC`, Internal serial `2026110504` and
External serial `2026110302` loaded on both BIND authorities. The dynamic
External zone was frozen only for the edit, validated and thawed; final
primary readback confirmed it unfrozen. The protected pre-change backup is
`/root/appear-core-dns-prechange-20261105T051040Z` on CT 25100.
`named.conf.local` was unchanged. Both Internal authoritative answers and
External answers through Cloudflare and Google matched the intended router
addresses.

CORE-ROUTER-CORDOBA has paired TCP 21051 destination-NAT and connection-mark
rules, a mark-restricted forward accept and an unmarked-dstnat guard. Existing
rules were preserved. The operator-approved legacy HTTP exception has no
source-address restriction; Digest authentication does not encrypt traffic.
SSH, HTTPS and Maintenance Center have no new publication. Authenticated native
configuration GETs succeeded through the hostname and public router address
from the development host; independent Internet-source acceptance was not run.
The assignment is recorded in the
[router endpoint registry](../MIKROTIK_MANAGER/config/public_nat_registry.yaml).

These forward records did not resolve APPEAR's reverse-lookup failure for
DNS gateway `192.168.15.1`. The subsequently approved PTR correction and
remaining resolver-cache blocker are described below. See the
[HEADEND DNS investigation](../../STREAMING/HEADEND/equipment/migration/reports/administrative_endpoints.md).

## Cordoba gateway Internal PTR (November 5, 2026)

The operator approved `192.168.15.1 PTR cordoba.intelisis.com.ar.` without a
new hostname or A-record change. The new Internal-only reverse zone
`15.168.192.in-addr.arpa` has serial `2026110501`, record/default/negative TTL
300, and the existing primary/secondary NS identities. It becomes authoritative
for this Internal /24; no other PTR owners were added.

Primary CT 25100 loaded the zone at `05:23:35 UTC`; secondary CT 26100 loaded
it at `05:23:39 UTC`. Both returned authoritative PTR answers. Existing
`dns-internal-transfer` TSIG, explicit notification and source-address
conventions were reused; secondary logs confirm a successful authenticated
transfer. The first notification preceded secondary configuration and returned
NOTAUTH; the subsequent transfer succeeded. Zone and full configuration checks
passed, and all pre-existing forward/reverse zone files retained their hashes.
Only the Internal reverse-zone declarations and the new zone data were added;
External DNS, existing A records and CORE NAT were unchanged. BIND was
reconfigured, not restarted.

Protected pre-change configurations remain on primary CT 25100 at
`/root/appear-gateway-ptr-prechange-20261105T052335Z` and secondary CT 26100 at
`/root/appear-gateway-ptr-prechange-20261105T052338Z`. Rollback removes only
the new Internal zone declaration and its dedicated file on each server,
validates the configuration and runs `rndc reconfig`; existing forward zones
need no serial change because no A record was changed.

The CORE resolver continued returning its cached NXDOMAIN. RouterOS
`/ip/dns/cache/all` showed the exact probe owner
`1.15.168.192.in-addr.arpa` with approximately 23 hours 49 minutes remaining.
The operator authorized exact-entry invalidation only; the attempted
`/ip/dns/cache/all/remove` returned `no such command`. No whole-cache flush,
static override, resolver change or restart was attempted. At that point APPEAR warning resolution remained blocked on the stale cache.

The operator subsequently authorized one full DNS cache flush on
CORE-ROUTER-CORDOBA. It completed at `05:26:47 UTC`; the gateway immediately
returned the correct PTR with `NOERROR`. The first APPEAR observation still
showed warning 4653071, but it was absent at `05:27:18 UTC` and at all three
subsequent observations through `05:28:48 UTC`. The warning cleared
automatically, with no manual alarm acknowledgement, device change or service
restart. This closes the observed DNS warning within the approved two-minute
verification window; ongoing recurrence monitoring remains appropriate.

## Related

- `../../INFRASTRUCTURE/PVE/` - Proxmox VE host infrastructure and guest provisioning
- `../MIKROTIK-MANAGER/` - Edge router fleet configuration and destination-NAT state
- `../../SERVICES/WEB/` - Ingress reverse-proxy services mapped from DNS endpoints
- `../../SERVICES/MAIL/` - Mailcow mail gateway DNS dependencies
