# Design: IBM FlashSystem monitoring via REST API + SNMP traps

**Created:** 2026-07-16
**Last updated:** 2026-07-17 (export format bumped to Zabbix 7.0; document translated to English)
**Status:** Approved / implemented
**Target platform:** Zabbix **7.0+** (7.0 / 7.2 / 7.4) · FlashSystem / Spectrum Virtualize **code level 8.5+**

## Problem

The previous template only monitored **status** (online/fault) and **per-pool capacity**
(MDiskGroup). Missing:

1. Storage-wide performance — throughput (MB/s) and IOPS.
2. Per-node / per-canister performance.
3. **Per-LUN (volume) space utilization**, to track growth and alert when a threshold is
   reached.

On top of that, the old architecture (a Go binary called as an *external script*, opening
**one SSH connection per item**) does not scale: an array with many objects generates
hundreds of SSH connections per polling cycle.

## Technical constraint discovered

**Live per-LUN performance** (IOPS/MBps of each volume) is **not** exposed by any CLI/REST
command in Spectrum Virtualize — those per-object stats only exist inside the XML statistics
files, collected via scp from the node. Decision: **leave per-LUN performance out of scope
for now.** System-wide and per-node performance (`lssystemstats` / `lsnodestats`) meets the
global-throughput need.

## Scope decisions

- Rewrite on the **native REST API** (HTTP), retiring the Go binary and the SSH-per-item model.
- Target environment: FlashSystem **code level 8.5+**, with access to real hardware for testing.
- Live per-LUN performance: **out of scope** in this phase.
- Hardware health/status and logical failures: via **SNMP traps** (asynchronous events),
  using `SVC_MIB_9.1.0.MIB`.
- Safety net: **one** lightweight REST health check (count of unfixed events), since traps
  only capture state changes.

## Architecture (hybrid)

### A) Metrics — REST API (polling)

The Spectrum Virtualize REST API (port 7443) is token-based: `POST /rest/auth` with
`X-Auth-Username` / `X-Auth-Password` → token; then `POST /rest/<command>` with
`X-Auth-Token`. Zabbix's HTTP agent item does not chain auth → use natively, so the
collectors are **Script (JavaScript) items** that, in a single run, authenticate and fire the
commands for the group, returning JSON. Individual values hang off as **dependent items** with
JSONPath preprocessing. Self-signed certificate → SSL verification is off by default.

**Master collectors (grouped by cadence, interval parameterized):**

| Collector | REST commands | Interval (macro / default) |
|---|---|---|
| Performance | `lssystemstats`, `lsnodestats` | `{$IBM.PERF.INTERVAL}` / **2m** |
| Capacity    | `lsvdisk` (bytes), `lsmdiskgrp`, `lseventlog` (heartbeat) | `{$IBM.CAP.INTERVAL}` / **10m** |

**Metrics delivered:**

- **System** (`lssystemstats`): read/write IOPS, read/write MB/s, read/write latency (ms),
  CPU %, cache %.
- **Per node/canister** (`lsnodestats`, via node LLD): same indicators per node.
- **Per LUN** (`lsvdisk`, via volume LLD): computed `used_capacity / capacity * 100` →
  utilization % + growth per volume.
- **Per pool** (`lsmdiskgrp`, via pool LLD): capacity, free, used %, status.
- **iSCSI throughput/IOPS** (system + per node): the `iscsi_mb` / `iscsi_io` stat names
  already returned by `lssystemstats` / `lsnodestats` — no extra command.
- **Per iSCSI port** (`lsportethernet`, via port LLD): negotiated Ethernet speed
  (`port_speed` → bps). `lsportethernet` replaces the deprecated `lsportip` on 8.4.2+ and is
  the one that reports the 10/25GbE optical adapter ports (`lsportip` returns the legacy
  onboard 1GbE ports).
- **Per battery** (`lsenclosurebattery`, via battery LLD): charge (`percent_charged`).
- **Health heartbeat** (`lseventlog -filtervalue fixed=no`): count of unfixed events, as a
  complementary sanity check to the traps.

**Metrics-only, no polled status (iSCSI transport):** this array is **iSCSI (IP-SAN)**, not
FC. Port and battery items are **historical metrics only** — link up/down, battery
status/EOL and other failures are reported via SNMP traps, so the REST items carry **no
status/failure triggers**. True **per-physical-port throughput** is not in the REST API (it
lives only in the XML stats files, like per-LUN performance); iSCSI throughput is therefore
delivered at the system/node level.

**Honesty note (fully-allocated volumes):** on *fully-allocated* volumes,
`used_capacity == capacity` (always 100%); real growth is only observable on
*thin/compressed* volumes. All are monitored, but the growth alert is meaningful on thin ones.

### B) Health/Status — SNMP traps (via MIB)

MIB root: `ibm2145TSVE = .1.3.6.1.4.1.2.6.190`. Three NOTIFICATION-TYPEs map directly to
severity:

| Trap | OID | Severity |
|---|---|---|
| `tsveETrap` (error) | `.1.3.6.1.4.1.2.6.190.1` | HIGH |
| `tsveWTrap` (warning) | `.1.3.6.1.4.1.2.6.190.2` | WARNING |
| `tsveITrap` (info) | `.1.3.6.1.4.1.2.6.190.3` | INFO |

Varbinds (`ibm2145TSVEObjects .4.N`) available for the message/trigger:
`tsveERRC` (error code, .4), `tsveOBJT` (object type, .11), `tsveOBJI` (object ID, .12),
`tsveOBJN` (object name, .17), `tsveNODE` (.8), `tsveERRS` (sequence, .9),
`tsveTIME` (.10), `tsveCLUS` (cluster, .7), `tsveMACH`/`tsveSERI` (machine/serial).

This **replaces** polling for drive, enclosure, battery, canister, PSU, array status and
volume failure — and removes the corresponding hardware LLDs.

**Template items:** 3 `snmptrap[...]` items matched by trap OID. `snmptrapd` renders the OID
symbolically (`enterprises.2.6.190.N`), so the match regex is `\.2\.6\.190\.(0\.)?N` — the tail
catches the symbolic form, the fully-numeric form, and the SNMPv1 `…190.0.N` form (optional
`(0\.)?`). The full trap text is kept in the item value (LOG). Each varbind arrives labeled as
`"# <Label> = <value>"` (e.g. `# Object Type = battery`, `# Error ID = …`, `# Node ID = 1`); the
trigger **Event name** and the `alert_type`/`error_code` **tags** are built with `regsub` on
those labels, so each problem is self-describing (type, object, code, node). Triggers:
Error→HIGH, Warning→WARNING, Info→INFO. Because traps are events (SVC does not send a reliable
clear), triggers use `manual_close: YES`; a trap item only re-evaluates its trigger when a new
trap arrives, so manual close is not defeated by stale data.

**Infrastructure requirements (documented in README, outside the template):**
- **SNMP** interface on the Zabbix host (traps match the host by IP, then the item by OID regex).
- Zabbix side: `snmptrapd` + `SNMPTrapperFile` configured; MIB loaded into `snmptrapd` for
  name resolution (optional).
- Array side: `mksnmpserver -ip <zabbix> -community <community>`.
- **ICMP ping** kept as an availability heartbeat.

## Runtime model (why JavaScript, and dependencies)

The Script collectors run on the **Duktape JavaScript engine embedded in the Zabbix
server/proxy binary** — in-process, not Node.js. Consequences:

- **No dependencies to install in the official `zabbix/zabbix-server-*` image.** The engine
  and the `HttpRequest` object (backed by the already-compiled libcurl) ship with the server.
- **No external JS packages** (`require`/npm do not exist here). Only built-in globals are
  available: `JSON`, `HttpRequest`, `Zabbix` (logging), `atob`/`btoa`.
- Engine is **ECMAScript 5.1-era**: no `let`/`const`, arrow functions or `fetch`. The
  collectors deliberately use `var`, function declarations and `HttpRequest`.
- **Python was rejected**: Zabbix has no native Python item type. Python would require an
  *external check* (a `python3` + `requests` install baked into a custom image), reintroduce a
  process-per-item fork/exec (the very scaling problem this rewrite removes), and pass
  credentials via argv (visible in `ps`). Keeping everything in the in-process Script item is
  strictly better for this use case.

## Macros

| Macro | Default | Use |
|---|---|---|
| `{$API_USER}` | — | REST user |
| `{$API_PASSWORD}` | — | REST password (Secret) |
| `{$API_PORT}` | `7443` | REST port |
| `{$IBM.CERT.VERIFY}` | `0` | SSL verification (self-signed; reserved) |
| `{$IBM.PERF.INTERVAL}` | `2m` | Performance collector interval |
| `{$IBM.CAP.INTERVAL}` | `10m` | Capacity collector interval |
| `{$SNMP_COMMUNITY}` | `public` | Trap community |
| `{$VOLUME_PCT_WARN}` | `80` | Per-LUN warning threshold (%) |
| `{$VOLUME_PCT_CRIT}` | `90` | Per-LUN critical threshold (%) |
| `{$POOL_PCT_WARN}` | `80` | Per-pool warning threshold (%) |
| `{$POOL_PCT_CRIT}` | `95` | Per-pool critical threshold (%) |

## Discovery (LLD)

Driven by the REST collectors' JSON:
- **Volumes** (`{#VOLUME_ID}`, `{#VOLUME_NAME}`, `{#POOL_NAME}`)
- **Nodes/canisters** (`{#NODE_ID}`, `{#NODE_NAME}`)
- **Pools** (`{#POOL_ID}`, `{#POOL_NAME}`)

Hardware LLDs (drives/enclosures/etc.) are removed — covered by traps.

## Components / units

1. **Performance collector** (Script JS) — auth + `lssystemstats`/`lsnodestats` → JSON.
2. **Capacity collector** (Script JS) — auth + `lsvdisk`/`lsmdiskgrp`/`lseventlog` → JSON.
3. **REST auth module** (JS `login`/`call` helpers, reused in both collectors).
4. **Dependent items + preprocessing** (JSONPath) — system, node, volume, pool metrics.
5. **LLD** — volumes, nodes, pools.
6. **SNMP trap items** (3) + operational-data mapping.
7. **Triggers** — capacity thresholds (macros) + trap severities + availability safety nets.
8. **Interface/macros/README** — SNMP infra and REST credentials.

## Error handling

- Auth/HTTP failure in the Script item → item unsupported; a "collector unavailable"
  (`nodata`) trigger avoids masking problems.
- Self-signed cert → `HttpRequest` does not verify the peer certificate (default). The
  `{$IBM.CERT.VERIFY}` macro is reserved for documentation/future use.
- Token expires on inactivity → each run re-authenticates (stateless), acceptable.
- Division by zero in "% per LUN" (capacity=0) → handled in the collector (returns 0).

## Testing

Incremental validation against the real 8.5+ array:
1. Auth + `lssystemstats` flow → confirm JSON and token.
2. Each collector in isolation → confirm the JSONPaths against real output before closing the
   block. Confirm the exact `stat_name`s (`vdisk_r_io`, `vdisk_w_io`, `vdisk_r_mb`,
   `vdisk_w_mb`, `vdisk_r_ms`, `vdisk_w_ms`, `cpu_pc`, `total_cache_pc`).
3. Volume/node/pool LLD → confirm macros are populated.
4. Traps: generate a test event on the array (`mksnmpserver` + event) → confirm reception and
   severity classification in Zabbix.
5. Thresholds → validate per-LUN and per-pool % trigger firing.

## Usage

The template is `ibm_flashsystem_template.yaml` (Zabbix **7.0** export format). Full operator
steps live in the repository **`README.md`**; the operational summary:

1. **Prepare the array (REST):** allow the Zabbix server/proxy to reach TCP **7443**; create a
   read-only monitoring user.
2. **Prepare traps:** on the Zabbix side, configure `snmptrapd` + `SNMPTrapperFile` /
   `StartSNMPTrapper=1`; on the array, `mksnmpserver -ip <zabbix_ip> -community <community>`.
3. **Import:** *Data collection → Templates → Import* → `ibm_flashsystem_template.yaml`.
4. **Link + configure the host:** link *IBM FlashSystem by HTTP*; add an **SNMP interface**
   (for traps) plus an address the collectors can reach; set the macros — at minimum
   `{$API_USER}`, `{$API_PASSWORD}`, and `{$SNMP_COMMUNITY}` to match the array.
5. **Verify:** *Test → Get value* on *Data collection: Performance* / *Capacity*; confirm
   discovery populates volumes/nodes/pools and that threshold triggers behave.

Adjust the dependent-item JSONPaths only if the real array reports different `stat_name`s or
field names (see Testing).

## Migration

- Remove the `src/` Go binary and `docker-compose.yaml` from the usage flow.
- Rewrite the template from scratch (this new model), exported as Zabbix 7.0.
- Keep the ICMP ping item.
- Update README: REST prerequisites (port 7443, user), SNMP trap infra (snmptrapd,
  mksnmpserver, MIB), and the new macros.

## Out of scope

- Live per-LUN performance (IOPS/MBps per volume) — would require an scp collector for the XML
  stats files. Possible future phase.
