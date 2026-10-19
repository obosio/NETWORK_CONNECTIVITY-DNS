# High-Port Scheme and DNS/CORE Reconciliation

**Observed:** October 16, 2026  
**Scope:** DNS answers, the workload high-port registry, related TIC migration
plan, and live configuration on the six CORE routers. Read-only review; no
DNS, router, firewall, or workload configuration was changed.

## Sources and interpretation

Terminology note: references in this report to the “external firewall” mean
the `/ip firewall filter` policy on the applicable CORE RouterOS router. There
is no separate external firewall appliance or control plane.

- Authenticated, read-only RouterOS API queries against CORE-ROUTER-CALINGASTA,
  CORE-ROUTER-CENTRO, CORE-ROUTER-CORDOBA, CORE-ROUTER-LUJAN,
  CORE-ROUTER-SAN-JUAN, and CORE-ROUTER-TIC. Queried `/ip service`, enabled
  `/ip firewall nat` destination-NAT rules, and `/ip firewall filter` input
  rules.
- DNS A/CNAME queries to the Internal resolver `192.168.25.100` and public
  recursive resolver `1.1.1.1`. Results below describe these observations, not
  guaranteed future state.
- The credential-free router-owned endpoint registry:
  [public_nat_registry.yaml](../../MIKROTIK_MANAGER/config/public_nat_registry.yaml)
- The separate TIC workload NAT plan:
  [tic_workload_migration_nat_plan.yaml](../../MIKROTIK_MANAGER/config/tic_workload_migration_nat_plan.yaml)
- The DNS-owned reconciliation ledger:
  [external_dns_reconciliation.yaml](../config/external_dns_reconciliation.yaml)
- DNS administration forwarding details:
  [DNS_PRIMARY_SECONDARY.md](../../../INFRASTRUCTURE/PVE/doc/DNS_PRIMARY_SECONDARY.md)

The registry is a plan and accounting source, not a DNS zone and not a live
router export. “Active” below means observed in enabled live destination-NAT
rules, with the registry/plan used for ownership and intended policy. DNS
records do not store TCP ports; clients must supply the assigned port, for
example `https://web.intelisis.com.ar:20602` or
`ssh -p 20603 web.intelisis.com.ar`.

## Port policy

This table records the allocator and observed router-management scheme used
for this October 16 read-only reconciliation. The authoritative service
classification and ingress-port policy is now consolidated in
[PVE's Service Ingress and NAT Port Policy](../../../INFRASTRUCTURE/PVE/doc/SERVICE_INGRESS_AND_NAT_POLICY.md).
The endpoint inventory below remains evidence of observed assignments, not
authorization to change or renumber them.

| Purpose | TCP range or formula | Notes |
| --- | --- | --- |
| Router management | `28000-28999` | SSH `28022`, Winbox `28291`, RouterOS API `28728`, API-SSL `28729`. |
| Workload Batch 1 | `20000 + (server_id * 10) + slot` | Slots 1–10; normally use the zero-padded destination IPv4 final octet for `server_id`. |
| Workload Batch 2 | `30000 + (server_id * 10) + slot` | Slots 1–10; check RouterOS dynamic services and tunnel listeners before allocating. |
| Workload slot meanings | Legacy allocations may use any slot; new private-use high-port allocations start at slot `4` | Public-use standard ports and private-use high ports are classified by intended audience, not DNS view. Existing assignments require an approved migration before renumbering. |

The management block and workload formula are distinct policies, but the
documented allocator does not specify a reserved-ID exclusion. Batch 1 IDs
`800-899` can generate ports in the router-management range; for example,
server ID `800` generates `28001-28010`. Do not allocate such IDs until the
policy and registry validation explicitly prevent this overlap. Batch 2 is
outside the `28xxx` block but still requires the documented listener-conflict
check.

High-port translation changes only the router-facing port. It does not change
the workload's native listener or itself establish source-address filtering.
For a hostname-based NAT connection, the DNS answer must lead the client to the
router address that owns the corresponding NAT rule. Direct-to-workload DNS
answers bypass that router-side mapping.

## Registry totals

| Registry state | Count | Contents |
| --- | ---: | --- |
| Active | 10 | Mantis 2; WEB 2; GPS SSH 1; IoT SSH 1; Cordoba S3 4. |
| Planned | 36 | PVE 24; PBS 3; PDM 3; Development SSH 2; TIC S3 4. |
| **Total** | **46** | Registry entries; not the count of all live high-port NAT rules. |

The TIC Gestion/Database mappings and DNS Webmin/SSH and Netmgr mappings are
live but are documented separately from this 46-entry public endpoint
registry. Including them, the six CORE routers have **38 enabled destination-
NAT rules** in TCP `20000-40000`, representing 20 logical port-to-workload
mappings when separate internal and external router addresses are counted
once.

## Active workload endpoints in the registry

| Service / FQDN | CORE router | External endpoint | Internal router endpoint | Backend | DNS Internal → External | Registry policy / qualification |
| --- | --- | --- | --- | --- | --- | --- |
| Mantis — `mantis.intelisis.com.ar` | TIC | `138.121.101.15:20502 -> 443`; `:20503 -> 22` | `10.8.0.61:20502 -> 443`; `:20503 -> 22` | `192.168.27.50` | `10.8.0.61` → CNAME `tic.intelisis.com.ar` → `138.121.101.15` | `public_https_external_firewall` / `public_ssh_external_firewall`; the TIC plan still says external acceptance is unverified, but the DNS service reconciliation records successful HTTPS/SSH transport checks and an authenticated login from this workstation on October 16. |
| WEB — `web.intelisis.com.ar` | Lujan | `138.121.100.15:20602 -> 443`; `:20603 -> 22` | `10.8.0.21:20602 -> 443`; `:20603 -> 22` | `192.168.25.60` | `10.8.0.21` → `138.121.100.15` | `public_https_external_firewall` / `public_ssh_external_firewall`. |
| GPS admin SSH — `gps.aero.com.ar` | Lujan | `138.121.100.2:20083 -> 22` | No matching internal-router NAT rule observed | `192.168.25.8` | `192.168.25.8` → `138.121.100.2` | `public_ssh_external_firewall`; the dedicated environment key was previously verified. Internal DNS points directly to the guest, not the NAT router. |
| IoT admin SSH — `iot.intelisis.com.ar` | Lujan | `138.121.100.6:20073 -> 8922` | No matching internal-router NAT rule observed | `192.168.25.7` | `192.168.25.7` → `138.121.100.6` | `public_ssh_external_firewall`; the registry owner is marked pending. Internal DNS points directly to the guest, not the NAT router. |
| Cordoba S3 — `s3-cor.intelisis.com.ar` | Cordoba | `45.238.8.15:20283 -> 22`; `:20284 -> 9443`; `:20285 -> 9444`; `:20286 -> 9445` | `10.8.0.51` on the same four ports | `192.168.24.28` | CNAME `cordoba.intelisis.com.ar` → `10.8.0.51` / `45.238.8.15` | SSH: `public_ssh_external_firewall`; HTTPS/admin/master/filer: registry says `public_https_unrestricted` while `audience` is `private`. Reconcile that policy-label conflict with the owner. This is distinct from TIC `s3.intelisis.com.ar`. |

All of the above mappings were present as enabled destination-NAT rules on the
listed CORE router(s) when inspected. The Mantis acceptance status is
inconsistent between the TIC plan and the later DNS reconciliation cited
above; the latter's test vantage does not establish reachability from every
client network. The external-firewall policies in the registry are
declarations, not policies independently verified by this audit.

## Other active high-port mappings outside the registry

| Service / FQDN | CORE router | External endpoint | Internal router endpoint | Backend | DNS Internal → External | Source/policy evidence |
| --- | --- | --- | --- | --- | --- | --- |
| Primary DNS admin — `ns1.intelisis.com.ar` | Lujan | `138.121.100.1:21002 -> 192.168.25.100:10000`; `:21003 -> :22` | `10.8.0.21:21002 -> 192.168.25.100:10000`; `:21003 -> :22` | CT 25100 | `10.8.0.21` → `138.121.100.1` | PVE documentation says source filtering is delegated to the external firewall; no source restriction is in these RouterOS NAT rules. |
| Secondary DNS admin — `ns2.intelisis.com.ar` | Centro | `138.121.102.1:21002 -> 192.168.26.100:10000`; `:21003 -> :22` | `10.8.0.11:21002 -> 192.168.26.100:10000`; `:21003 -> :22` | CT 26100 | `10.8.0.11` → `138.121.102.1` | PVE documentation says source filtering is delegated to the external firewall; no source restriction is in these RouterOS NAT rules. |
| Netmgr — `netmgr.intelisis.com.ar` | Lujan | `138.121.100.15:20702 -> 192.168.25.70:443`; `:20703 -> :22` | `10.8.0.21:20702 -> 192.168.25.70:443`; `:20703 -> :22` | `192.168.25.70` | CNAME `lujan.intelisis.com.ar` → `10.8.0.21` / `138.121.100.15` | Not listed in the 46-entry registry; source policy was not independently reviewed. |
| Gestion — `gestion.intelisis.com.ar` | TIC | `138.121.101.15:20202 -> 192.168.27.20:443`; `:20203 -> :22` | `10.8.0.61:20202 -> 192.168.27.20:443`; `:20203 -> :22` | `192.168.27.20` | CNAME `tic.intelisis.com.ar` → `10.8.0.61` / `138.121.101.15` | Separate TIC plan; source policy owner is the external firewall. External HTTPS test in that plan timed out from its test origin. |
| Database — `db1.intelisis.com.ar` | TIC | `138.121.101.15:20253 -> 192.168.27.25:22`; `:20254 -> :5432` | `10.8.0.61:20253 -> 192.168.27.25:22`; `:20254 -> :5432` | `192.168.27.25` | `192.168.27.25` → `138.121.101.15` | Separate TIC plan; CORE RouterOS filter source policy was not independently verified. External route acceptance was waived and remains unverified. Internal DNS points directly to the guest, so a client using `db1.intelisis.com.ar:<high-port>` will not hit the internal-router NAT. |

The DNS primary/secondary and Netmgr high ports are present on both the
internal and external router addresses. TIC Gestion and Database also have
both internal and external NAT rules. These mappings were not counted in the
46-entry registry total.

## Planned registry endpoints

The following 30 PVE/PBS/PDM records are `planned` in the registry. The port
and backend columns reproduce the registered mapping; none was observed as an
active high-port destination-NAT rule on the six CORE routers.

| Site / public edge | Port | Registered FQDN | Backend destination |
| --- | ---: | --- | --- |
| Centro / `138.121.102.15` | 20112 | `pve-upr-1-cen.intelisis.com.ar` | `192.168.11.11:8006` |
| Centro / `138.121.102.15` | 20122 | `pve-upr-2-cen.intelisis.com.ar` | `192.168.11.12:8006` |
| Centro / `138.121.102.15` | 20132 | `pve-upr-3-cen.intelisis.com.ar` | `192.168.11.13:8006` |
| Centro / `138.121.102.15` | 20212 | `pve-lwr-1-cen.intelisis.com.ar` | `192.168.11.21:8006` |
| Centro / `138.121.102.15` | 20222 | `pve-lwr-2-cen.intelisis.com.ar` | `192.168.11.22:8006` |
| Centro / `138.121.102.15` | 20232 | `pve-lwr-3-cen.intelisis.com.ar` | `192.168.11.23:8006` |
| Centro / `138.121.102.15` | 20312 | `pve-00-cen.intelisis.com.ar` | `192.168.11.31:8006` |
| Centro / `138.121.102.15` | 20314 | `pbs-00-cen.intelisis.com.ar` | `192.168.11.31:8007` |
| Centro / `138.121.102.15` | 20315 | `pdm-00-cen.intelisis.com.ar` | `192.168.11.31:8443` |
| Lujan / `138.121.100.15` | 20112 | `pve-upr-1-luj.intelisis.com.ar` | `192.168.12.11:8006` |
| Lujan / `138.121.100.15` | 20122 | `pve-upr-2-luj.intelisis.com.ar` | `192.168.12.12:8006` |
| Lujan / `138.121.100.15` | 20132 | `pve-upr-3-luj.intelisis.com.ar` | `192.168.12.13:8006` |
| Lujan / `138.121.100.15` | 20212 | `pve-lwr-1-luj.intelisis.com.ar` | `192.168.12.21:8006` |
| Lujan / `138.121.100.15` | 20222 | `pve-lwr-2-luj.intelisis.com.ar` | `192.168.12.22:8006` |
| Lujan / `138.121.100.15` | 20232 | `pve-lwr-3-luj.intelisis.com.ar` | `192.168.12.23:8006` |
| Lujan / `138.121.100.15` | 20312 | `pve-00-luj.intelisis.com.ar` | `192.168.12.31:8006` |
| Lujan / `138.121.100.15` | 20314 | `pbs-00-luj.intelisis.com.ar` | `192.168.12.31:8007` |
| Lujan / `138.121.100.15` | 20315 | `pdm-luj.intelisis.com.ar` | `192.168.12.31:8443` |
| Cordoba / `45.238.8.15` | 20112 | `pve-upr-1-cor.intelisis.com.ar` | `192.168.15.11:8006` |
| Cordoba / `45.238.8.15` | 20122 | `pve-upr-2-cor.intelisis.com.ar` | `192.168.15.12:8006` |
| Cordoba / `45.238.8.15` | 20132 | `pve-upr-3-cor.intelisis.com.ar` | `192.168.15.13:8006` |
| Cordoba / `45.238.8.15` | 20212 | `pve-lwr-1-cor.intelisis.com.ar` | `192.168.15.21:8006` |
| Cordoba / `45.238.8.15` | 20222 | `pve-lwr-2-cor.intelisis.com.ar` | `192.168.15.22:8006` |
| Cordoba / `45.238.8.15` | 20232 | `pve-lwr-3-cor.intelisis.com.ar` | `192.168.15.23:8006` |
| Cordoba / `45.238.8.15` | 20312 | `pve-00-cor.intelisis.com.ar` | `192.168.15.31:8006` |
| Cordoba / `45.238.8.15` | 20314 | `pbs-00-cor.intelisis.com.ar` | `192.168.15.31:8007` |
| Cordoba / `45.238.8.15` | 20315 | `pdm-cor.intelisis.com.ar` | `192.168.15.31:8443` |
| TIC / `138.121.101.15` | 20112 | `pve-01-tic.intelisis.com.ar` | `192.168.16.11:8006` |
| TIC / `138.121.101.15` | 20122 | `pve-02-tic.intelisis.com.ar` | `192.168.16.12:8006` |
| TIC / `138.121.101.15` | 20132 | `pve-03-tic.intelisis.com.ar` | `192.168.16.13:8006` |

All PVE/PBS/PDM hostnames queried returned `NXDOMAIN` at `1.1.1.1`. Existing
Internal records resolve to workload addresses (PBS/PDM records may be CNAMEs
to the PVE host), not to the edge router addresses required for the proposed
high-port NAT. Therefore, these entries need both DNS-target and publication
decisions before activation; the current Internal records do not route
high-port connections through the registered NAT.

The PDM FQDNs also differ between the registry and Internal DNS:

| Site | Registry FQDN | Internal DNS name observed | Reconciliation |
| --- | --- | --- | --- |
| Centro | `pdm-00-cen.intelisis.com.ar` | `pdm-00-cen.intelisis.com.ar` | Name matches; it is a CNAME to `pve-00-cen`. |
| Lujan | `pdm-luj.intelisis.com.ar` | `pdm-00-luj.intelisis.com.ar` | Registry FQDN returns NXDOMAIN internally; the existing `pdm-00-luj` record is a CNAME to `pve-00-luj`. |
| Cordoba | `pdm-cor.intelisis.com.ar` | `pdm-00-cor.intelisis.com.ar` | Registry FQDN returns NXDOMAIN internally; the existing `pdm-00-cor` record is a CNAME to `pve-00-cor`. |

### Other planned entries

| Site / public edge | Port | Registered FQDN | Backend destination | DNS observation |
| --- | ---: | --- | --- | --- |
| TIC / `138.121.101.15` | 20153 | `development.intelisis.com.ar` | `192.168.27.15:22` | Current Internal A is `192.168.27.15`; External A is `138.121.101.8`, not the registered TIC edge. |
| TIC / `138.121.101.15` | 20653 | `development.intelisis.com.ar` | `192.168.27.65:22` | Same DNS answers as above; target `.27.65` is a proposal, not the current DNS destination. |
| TIC / `138.121.101.15` | 20282 | `s3.intelisis.com.ar` | `192.168.27.28:443` | Internal A is `192.168.27.28`; External is `NXDOMAIN`. |
| TIC / `138.121.101.15` | 20284 | `s3.intelisis.com.ar` | `192.168.27.28:9443` | Same DNS answers; planned only. |
| TIC / `138.121.101.15` | 20285 | `s3.intelisis.com.ar` | `192.168.27.28:9444` | Same DNS answers; planned only. |
| TIC / `138.121.101.15` | 20286 | `s3.intelisis.com.ar` | `192.168.27.28:9445` | Same DNS answers; planned only. |

For Development, proposal documentation describes a transitional
`138.121.101.8:20022 -> 192.168.27.15:22` path. The current DNS answer confirms
`.101.8`, but no matching active TCP `20022` destination-NAT rule was found on
the six CORE routers in this audit. The endpoint could be outside those six
router configurations; its live owner/router path was not established here.
Do not treat either registry proposal as a verified replacement.

TIC S3 DNS analysis remains deferred until the external load-balancer
deployment is complete. The separate Cordoba S3 mappings in the active table
remain live; this report makes no S3 DNS or NAT change.

## CORE router service and NAT matrix

| CORE router | SSH | Winbox | API | API-SSL | API-SSL certificate | Router `www` service | Enabled high-port DNAT rules |
| --- | ---: | ---: | ---: | ---: | --- | --- | ---: |
| Calingasta | 28022 | 28291 | 28728 | 28729 | `none` | 8081 enabled; `www-ssl` disabled | 0 |
| Centro | 28022 | 28291 | 28728 | 28729 | `none` | 20080 enabled; `www-ssl` disabled | 4 |
| Cordoba | 28022 | 28291 | 28728 | 28729 | `none` | `www` and `www-ssl` disabled | 8 |
| Lujan | 28022 | 28291 | 28728 | 28729 | `none` | 20080 enabled; `www-ssl` disabled | 14 |
| San Juan | 28022 | 28291 | 28728 | 28729 | `none` | 8081 enabled; `www-ssl` disabled | 0 |
| TIC | 28022 | 28291 | 28728 | 28729 | `none` | `www` and `www-ssl` disabled | 12 |

The four management ports are enabled and match the `28xxx` standard on all
six CORE routers. API-SSL has no certificate configured on any of them, so the
configured port does not establish usable TLS. The API service query returned
two enabled `api` rows on the same port on each router. This is recorded as an
observation only; this audit did not determine whether RouterOS/API output
semantics or configuration state explain the duplicate rows.

The live high-port destination-NAT distribution is:

| Router | Live high-port services |
| --- | --- |
| Calingasta | None observed. |
| Centro | DNS secondary Webmin/SSH: `21002`, `21003`, each on internal and external router addresses. |
| Cordoba | S3: `20283-20286`, each on internal and external router addresses. |
| Lujan | DNS primary Webmin/SSH `21002-21003`; WEB `20602-20603`; Netmgr `20702-20703`, each on internal and external router addresses; IoT `20073` and GPS `20083` on their dedicated public addresses only. |
| San Juan | None observed. |
| TIC | Gestion `20202-20203`, Database `20253-20254`, and Mantis `20502-20503`, each on internal and external router addresses. |

### Provider-facing input-rule observations

The following are the explicit TCP destination-port lists observed in the
provider-interface input accept rules; they are not a complete reachability
assessment.

| CORE router | Explicit provider TCP allow-list |
| --- | --- |
| Calingasta | `1723,8081,28291,8221,8222` |
| Centro | `1723,20080,20291,8221,8222` |
| Cordoba | `1723,8081,20291,8221,8222` |
| Lujan | `1723,28081,20291,28221,28222` |
| San Juan | `1723,8081,28291,8221,8222` |
| TIC | `1723,8081,20291,8221,8222` |

Only Calingasta and San Juan explicitly list the target Winbox port `28291`;
the other four list `20291` instead. None of these explicit lists names SSH
`28022`, API `28728`, or API-SSL `28729`. However, an enabled provider-interface
TCP accept rule without a `dst-port` match was also returned on each CORE
router. The TIC provider-interface `Drop` rule observed in this query was
disabled. Consequently, the explicit-port lists alone do not establish
effective service reachability or exposure. A full ordered filter, interface,
address, and source-policy review would be required for that conclusion.

## Reconciliation findings and gates

1. **Protect the management range in the workload allocator.** Define and
   validate an explicit exclusion before using workload IDs that can generate
   `28xxx` Batch 1 ports.
2. **Align DNS with router NAT endpoints before enabling plans.** The PVE/PBS/
   PDM internal names currently point to workloads, not edge routers; their
   external answers are NXDOMAIN. The registered Lujan and Cordoba PDM FQDNs
   also do not match the existing Internal aliases.
3. **Resolve Development's existing-path discrepancy.** The public DNS answer
   is `.101.8`, the documented legacy endpoint uses `.101.8:20022`, and the
   registry proposals use `.101.15:20153` and `.101.15:20653`. No active
   `20022` destination-NAT rule was found on the six inspected CORE routers.
   Verify the actual owner/router path and target before treating a proposal
   as a cutover.
4. **Reconcile Cordoba S3 audience and policy labels.** The active registry
   entries for ports `20284-20286` label the audience `private` but use
   `public_https_unrestricted` as the source policy. Confirm the intended
   external-firewall policy with the service owner. Do not confuse these
   active Cordoba records with the deferred TIC S3/load-balancer work.
5. **Investigate RouterOS API inventory anomalies.** Confirm the duplicate
   `api` rows and install/verify certificates if API-SSL is expected to be
   usable. No service changes were made.
6. **Review router input policy separately.** The explicit provider allow
   lists do not consistently reflect the configured management ports; broad
   TCP accepts and the disabled TIC drop mean these observations must not be
   interpreted as proof of reachability or non-reachability. Lujan's `www`
   service listens on `20080`, while its explicit provider TCP list contains
   `28081` instead.
7. **Keep second-batch conflict checks as an activation gate.** No
   comprehensive dynamic-service/tunnel listener audit is claimed here.

## Conclusion

The six CORE routers' configured SSH, Winbox, API, and API-SSL ports match the
`28xxx` target, but API-SSL is not TLS-ready based on the observed
`certificate=none` values. Live application high-port NAT consists of 38
rules, while only 10 of the 46 registry endpoints are active; the remaining
36 are plans. DNS is aligned with the edge NAT for WEB, Mantis, Netmgr,
Gestion, the DNS administration endpoints, and Cordoba S3. GPS and IoT use
their dedicated public addresses for high-port SSH, while their Internal DNS
answers point directly to the workloads. Database and several planned
infrastructure, Development, and TIC S3 names need explicit DNS/route
reconciliation before their high-port records can be considered usable.
