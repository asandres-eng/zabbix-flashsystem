# Zabbix template — IBM FlashSystem (REST API + SNMP traps)

Monitoring template for IBM FlashSystem / Spectrum Virtualize (code level **8.5+**) built on
the native **REST API** (port 7443) for metrics and **SNMP traps** for health/status events.
It replaces the previous SSH/Go external-script approach (one SSH connection per item), which
did not scale.

- **Metrics (REST polling):** system and per-node performance (IOPS, throughput, latency,
  CPU, cache — including per-node/system **iSCSI** throughput and IOPS), per-volume space
  utilization, per-pool capacity, per-iSCSI-port **negotiated Ethernet speed**, per-battery
  **charge**, and an unfixed-events health heartbeat. Two Script (JavaScript) collectors
  authenticate once per run and fan the result out to dependent items via JSONPath.
- **Health/status (SNMP traps):** error / warning / informational traps (per `SVC_MIB_9.1.0.MIB`)
  mapped to trigger severities. Hardware/link failures (battery status/EOL, port link
  up/down, etc.) are covered **here**, not by polling — the REST items above are
  historical-metric-only and carry no failure/status triggers.
- **Dashboards:** *Performance* (system graphs + a per-node page where each chart overlays
  **all nodes** — IOPS read/write, iSCSI IOPS, throughput read/write, iSCSI throughput), *Volumes (LUN
  administration)* (utilization honeycomb + per-volume growth), *Storage administration*
  (pool capacity, health tiles, battery charge, iSCSI port negotiated-speed honeycomb). The
  port speed has no trend graph (the value is near-constant) — a speed change or link-down is
  reported by the SNMP error/warning traps, not by polling.
- **Out of scope:** live per-LUN performance (IOPS/MBps per volume) and live **per-physical-port
  iSCSI throughput** — both only exist in the XML statistics files (scp), not in the REST API.
  iSCSI throughput is therefore delivered at the system/node level (`iscsi_mb`/`iscsi_io`).

Files:

```
ibm_flashsystem_template.yaml   # the template (Zabbix 7.0 export format; imports on 7.0/7.2/7.4)
SVC_MIB_9.1.0.MIB               # trap reference / snmptrapd name resolution
README.md
docs/design/                    # design + implementation plan
```

## Prerequisites

### 1. REST API (metrics)

1. On the network, allow the Zabbix server/proxy to reach the storage on TCP **7443**.
2. On the array, create a monitoring user with read (Monitor role is enough) access.
3. Self-signed certificates are expected. The Script `HttpRequest` object does not verify
   peer certificates, so no CA import is required. `{$IBM.CERT.VERIFY}` is reserved for the
   future and defaults to `0`.

### 2. SNMP traps (health/status)

1. On the **Zabbix side**, configure `snmptrapd` and set `SNMPTrapperFile` /
   `StartSNMPTrapper=1` in `zabbix_server.conf`. Optionally load `SVC_MIB_9.1.0.MIB` in
   `snmptrapd` for symbolic name resolution.
2. On the **array**, register the Zabbix trap receiver:
   `mksnmpserver -ip <zabbix_ip> -community <community>`.
3. On the Zabbix host, add an **SNMP interface** (traps are matched to the host by source IP,
   then to items by OID regex). Set `{$SNMP_COMMUNITY}` to match.

Trap OIDs (root `ibm2145TSVE = .1.3.6.1.4.1.2.6.190`):

| Trap | OID | Item / severity |
|---|---|---|
| `tsveETrap` (error) | `.1.3.6.1.4.1.2.6.190.1` | Event: Error trap — HIGH |
| `tsveWTrap` (warning) | `.1.3.6.1.4.1.2.6.190.2` | Event: Warning trap — WARNING |
| `tsveITrap` (info) | `.1.3.6.1.4.1.2.6.190.3` | Event: Informational trap — INFO |

Useful varbinds (`.190.4.N`): error code `.4`, cluster `.7`, node `.8`, sequence `.9`,
time `.10`, object type `.11`, object id `.12`, object name `.17`. The full trap text is kept
in the item value and surfaced in the trigger operational data. Trap triggers use
`manual_close: YES` (Spectrum Virtualize does not send a reliable clear).

> **Trap matching (validated against a real 9.1.0 trap).** `snmptrapd` renders the trap OID
> **symbolically** as `enterprises.2.6.190.N` (not fully numeric), so the `snmptrap[...]` item
> regex is `\.2\.6\.190\.(0\.)?N`. That tail matches the symbolic form, the fully-numeric form
> (`…2.6.190.N`) **and** the SNMPv1 form (`…190.0.N`, via the optional `(0\.)?`). N = 1 error /
> 2 warning / 3 info — each trap matches exactly one item.
>
> **Readable problems.** The array packs each varbind as `"# <Label> = <value>"` (e.g.
> `# Object Type = battery`, `# Error ID = 1135 : …`, `# Node ID = 1`). The trigger **Event
> name** and the `alert_type` / `error_code` **tags** are built with `regsub` on those labels,
> so a problem reads e.g. `FlashSystem error: battery - 1135 : Battery fault (node 1)` with
> `alert_type=battery`. Tune the `regsub` labels only if a future code level changes the text.

## Install

1. In Zabbix (**7.0+**): *Data collection → Templates → Import* → `ibm_flashsystem_template.yaml`.
2. Create/select the host, link **IBM FlashSystem by HTTP**, add an **SNMP interface** (for
   traps) and an **Agent/DNS/IP** the collectors can reach.
3. Fill in the macros (below).

## Macros

| Macro | Default | Use |
|---|---|---|
| `{$API_USER}` | — | REST API user |
| `{$API_PASSWORD}` | — | REST API password (Secret) |
| `{$API_PORT}` | `7443` | REST API port |
| `{$IBM.CERT.VERIFY}` | `0` | SSL verification (reserved; self-signed) |
| `{$IBM.PERF.INTERVAL}` | `2m` | Performance collector interval |
| `{$IBM.CAP.INTERVAL}` | `10m` | Capacity collector interval |
| `{$SNMP_COMMUNITY}` | `public` | Trap community |
| `{$VOLUME_PCT_WARN}` | `80` | Per-volume warning threshold (%) |
| `{$VOLUME_PCT_CRIT}` | `90` | Per-volume critical threshold (%) |
| `{$POOL_PCT_WARN}` | `80` | Per-pool warning threshold (%) |
| `{$POOL_PCT_CRIT}` | `95` | Per-pool critical threshold (%) |

## Validation against the real array

The collectors depend on the exact `stat_name` values returned by `lssystemstats` /
`lsnodestats` and the field names of `lsvdisk` / `lsmdiskgrp`. Confirm them on the target
8.5+ array and adjust the JSONPaths if needed:

1. **Import** the YAML without schema errors.
2. **Auth + performance:** *Test → Get value* on *Data collection: Performance*; confirm a
   token is returned and the `stat_name`s (`vdisk_r_io`, `vdisk_w_io`, `vdisk_r_mb`,
   `vdisk_w_mb`, `vdisk_r_ms`, `vdisk_w_ms`, `cpu_pc`, `total_cache_pc`, and the iSCSI ones
   `iscsi_mb` / `iscsi_io`) match the dependent item JSONPaths.
3. **Capacity:** confirm `capacity`, `used_capacity`, `free_capacity`, `mdisk_grp_name`,
   `status`, and whether `{ "bytes": true }` is accepted (`toBytes` tolerates both numeric and
   unit-suffixed strings either way).
4. **iSCSI ports & batteries:** confirm `lsportethernet` (replaces the deprecated `lsportip`
   on 8.4.2+; needed for the 10/25GbE optical ports) exposes `node_id` / `port_id` /
   `port_speed` (negotiated speed, e.g. `10Gb/s`; `parseSpeed` converts to bps — the collector
   also falls back to a `speed` field and to `node_id` when `node_name` is absent), and that
   `lsenclosurebattery` exposes `enclosure_id` / `battery_id` / `percent_charged`.
5. **LLD:** confirm nodes, volumes, pools, iSCSI ports and batteries are discovered and
   prototypes populate.
6. **Thresholds:** confirm the per-volume and per-pool `%` triggers fire. (Ports and batteries
   are metric-only — no triggers by design; failures come via traps.)
7. **Traps:** generate a test event on the array (or send a synthetic trap), confirm
   reception in `snmptrapd`, item matching and severity classification. Quick synthetic tests
   (the trap must arrive from the host's SNMP-interface IP to match the host — temporarily
   point the interface at the sending host if needed):
   - SNMPv1 (IBM default → OID becomes `…190.0.1`):
     `snmptrap -v1 -c <community> <zabbix_ip> .1.3.6.1.4.1.2.6.190 <src_ip> 6 1 '' .1.3.6.1.4.1.2.6.190.4.4 s "1234"`
   - SNMPv2c (`…190.1`):
     `snmptrap -v2c -c <community> <zabbix_ip> '' .1.3.6.1.4.1.2.6.190.1 .1.3.6.1.4.1.2.6.190.4.4 s "1234"`
   Both must land on *Event: Error trap* and raise the HIGH trigger.

> **Note (fully-allocated volumes):** for fully-allocated volumes `used_capacity == capacity`
> (always 100%); real growth is only observable on thin/compressed volumes. All are monitored,
> but the growth alert is meaningful on thin volumes.
