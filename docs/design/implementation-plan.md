# Plano de implementação — Template Zabbix IBM FlashSystem (REST API + SNMP Traps)

## Context (por que)

O template Zabbix atual do IBM FlashSystem monitora só **status** (online/fault) e
**capacidade por pool**, através de um binário Go chamado como *external script* que abre
**uma conexão SSH por item** — não escala e não cobre performance nem consumo por LUN.

O usuário precisa de: (1) performance do storage como um todo (IOPS e MB/s), (2)
performance por nó, e (3) **consumo percentual por LUN** para acompanhar crescimento e
alertar em threshold. Decisão de arquitetura tomada no brainstorming: **reescrever do zero
usando a REST API nativa do Zabbix (HTTP)**, aposentando o Go/SSH, e usar **SNMP traps**
(via `SVC_MIB_9.1.0.MIB`) para eventos de saúde/status. Performance ao vivo **por-LUN** fica
**fora de escopo** (só existe em arquivos XML de stats via scp).

Design completo aprovado em: `docs/design/2026-07-16-flashsystem-rest-snmp-monitoring-design.md`.

> **Nota sobre este arquivo:** a sessão de desenvolvimento é nova/dedicada. O **passo 0** é
> criar a pasta limpa e salvar este plano como `docs/design/implementation-plan.md` dentro
> dela, para ficar versionado no projeto.

---

## Passo 0 — Setup da pasta limpa (fresh start)

Ainda **não existe** repositório novo. Criar do zero:

```bash
NEW=/Users/andrus/Documents/TRT14/flashsystem/zabbix-flashsystem-template
OLD=/Users/andrus/Documents/TRT14/flashsystem/zabbix-ibm-flashsystem-template
mkdir -p "$NEW/docs/design"
cp "$OLD/SVC_MIB_9.1.0.MIB" "$NEW/"
cp "$OLD/docs/design/2026-07-16-flashsystem-rest-snmp-monitoring-design.md" "$NEW/docs/design/"
cd "$NEW"
git init
git config user.email "andrussandres@gmail.com"
git config user.name "andrus"     # ajustar se quiser outro nome de autor
```

- **Salvar este plano** em `$NEW/docs/design/implementation-plan.md`.
- **Reabrir o Claude Code nessa pasta** (`cd $NEW && claude`) para que a sessão de dev tenha
  a pasta como raiz.
- **Commits SEM qualquer co-autoria/atribuição de IA** (nem trailers `Co-Authored-By`, nem
  `Claude-Session`). Já verificado que não há traços na história atual.
- Pasta antiga (`zabbix-ibm-flashsystem-template`) pode ser removida pelo usuário depois.

Arquivos finais do projeto novo:
```
zabbix-flashsystem-template/
├── ibm_flashsystem_template.yaml        # o template reescrito
├── SVC_MIB_9.1.0.MIB                     # referência dos traps
├── README.md                             # instruções novas
├── .gitignore
└── docs/design/
    ├── 2026-07-16-...-design.md
    └── implementation-plan.md            # este plano
```

---

## Passo 1 — Estrutura do template (`ibm_flashsystem_template.yaml`)

Formato de export **Zabbix 6.4** (`zabbix_export.version: '6.4'`). Nome do template:
`IBM FlashSystem by HTTP`. Grupos: `Templates/SAN`. Cada entidade (template, itens,
triggers, discovery rules, prototypes, macros) precisa de **UUID** de 32 hex — gerar com
`uuidgen | tr -d '-' | tr 'A-Z' 'a-z'`.

### 1a. Macros (com defaults)

| Macro | Default | Uso |
|---|---|---|
| `{$API_USER}` | — | Usuário da REST API |
| `{$API_PASSWORD}` | — | Senha (tipo Secret) |
| `{$API_PORT}` | `7443` | Porta REST |
| `{$IBM.CERT.VERIFY}` | `0` | Verificação SSL (cert self-signed) |
| `{$IBM.PERF.INTERVAL}` | `2m` | Intervalo do coletor de performance |
| `{$IBM.CAP.INTERVAL}` | `10m` | Intervalo do coletor de capacidade |
| `{$SNMP_COMMUNITY}` | `public` | Community dos traps |
| `{$VOLUME_PCT_WARN}` | `80` | Warning % por LUN |
| `{$VOLUME_PCT_CRIT}` | `90` | Crítico % por LUN |
| `{$POOL_PCT_WARN}` | `80` | Warning % por pool |
| `{$POOL_PCT_CRIT}` | `95` | Crítico % por pool |

### 1b. Convenção de nomes (pedido explícito: intuitivos e organizados)

Prefixo por componente + `[{#NOME}]` nas LLD:
- **Sistema:** `System: Read IOPS`, `System: Write IOPS`, `System: Read throughput`,
  `System: Write throughput`, `System: Read latency`, `System: Write latency`,
  `System: CPU utilization`, `System: Cache utilization`
- **Nó:** `Node [{#NODE_NAME}]: Read IOPS`, `... Write IOPS`, `... Read throughput`, etc.
- **Volume:** `Volume [{#VOLUME_NAME}]: Space utilization`, `... Capacity`, `... Used space`
- **Pool:** `Pool [{#POOL_NAME}]: Space utilization`, `... Capacity`, `... Free space`, `... Status`
- **Coletores (master):** `Data collection: Performance`, `Data collection: Capacity`
- **Eventos (traps):** `Event: Error trap`, `Event: Warning trap`, `Event: Informational trap`
- **Saúde:** `Health: Unfixed events`
- **Disponibilidade:** `Availability: ICMP ping`

**Tags** por item para organização/dashboards: `component: system|node|volume|pool|event|health`
e `scope: performance|capacity|availability`.

---

## Passo 2 — Coletores REST (itens tipo Script / JavaScript)

Os parâmetros do item Script chegam ao JS como o objeto `value` (`JSON.parse(value)`).
Cada coletor autentica (`POST /rest/auth` → token) e dispara seus comandos
(`POST /rest/<cmd>` com header `X-Auth-Token`), devolvendo **um JSON** consumido por
dependent items.

### 2a. `Data collection: Performance` (intervalo `{$IBM.PERF.INTERVAL}`)
Parameters: `host={HOST.CONN}`, `port={$API_PORT}`, `user={$API_USER}`, `password={$API_PASSWORD}`.

```javascript
var p = JSON.parse(value);
var base = 'https://' + p.host + ':' + p.port + '/rest';

function login() {
    var r = new HttpRequest();
    r.addHeader('Content-Type: application/json');
    r.addHeader('X-Auth-Username: ' + p.user);
    r.addHeader('X-Auth-Password: ' + p.password);
    var resp = r.post(base + '/auth');
    if (r.getStatus() !== 200) throw 'auth HTTP ' + r.getStatus() + ': ' + resp;
    return JSON.parse(resp).token;
}
function call(token, cmd, body) {
    var r = new HttpRequest();
    r.addHeader('Content-Type: application/json');
    r.addHeader('X-Auth-Token: ' + token);
    var resp = r.post(base + '/' + cmd, body ? JSON.stringify(body) : '{}');
    if (r.getStatus() !== 200) throw cmd + ' HTTP ' + r.getStatus() + ': ' + resp;
    return JSON.parse(resp);
}

var token = login();

var sys = call(token, 'lssystemstats');            // [{stat_name, stat_current, ...}]
var system = {};
for (var i = 0; i < sys.length; i++) system[sys[i].stat_name] = sys[i].stat_current;

var nr = call(token, 'lsnodestats');               // linhas (node x stat)
var nmap = {};
for (var j = 0; j < nr.length; j++) {
    var x = nr[j];
    if (!nmap[x.node_id]) nmap[x.node_id] = { node_id: x.node_id, node_name: x.node_name, stats: {} };
    nmap[x.node_id].stats[x.stat_name] = x.stat_current;
}
var nodes = [];
for (var k in nmap) nodes.push(nmap[k]);

return JSON.stringify({ system: system, nodes: nodes });
```

### 2b. `Data collection: Capacity` (intervalo `{$IBM.CAP.INTERVAL}`)
Mesmas funções `login`/`call` + helper `toBytes` (tolera valor numérico OU string com
unidade tipo `10.04TB`, caso o flag `bytes` não seja aceito pela API):

```javascript
function toBytes(v) {
    if (v === undefined || v === null || v === '') return 0;
    var s = ('' + v).trim();
    var m = s.match(/^([0-9.]+)\s*([KMGTP]?)B?$/i);
    if (!m) { var n = parseFloat(s); return isNaN(n) ? 0 : n; }
    var mult = {'':1, K:1024, M:1048576, G:1073741824, T:1099511627776, P:1125899906842624};
    return parseFloat(m[1]) * (mult[m[2].toUpperCase()] || 1);
}
var token = login();

var vd = call(token, 'lsvdisk', { bytes: true });
var volumes = [];
for (var i = 0; i < vd.length; i++) {
    var v = vd[i], cap = toBytes(v.capacity), used = toBytes(v.used_capacity);
    volumes.push({ id: v.id, name: v.name, pool: v.mdisk_grp_name || '',
        capacity: cap, used: used, pct: cap > 0 ? used / cap * 100 : 0, status: v.status });
}
var gp = call(token, 'lsmdiskgrp', { bytes: true });
var pools = [];
for (var g = 0; g < gp.length; g++) {
    var m = gp[g], pc = toBytes(m.capacity), pf = toBytes(m.free_capacity);
    pools.push({ id: m.id, name: m.name, capacity: pc, free: pf,
        pct: pc > 0 ? (pc - pf) / pc * 100 : 0, status: m.status });
}
var ev = call(token, 'lseventlog', { filtervalue: 'fixed=no' });

return JSON.stringify({ volumes: volumes, pools: pools, unfixed_events: ev.length });
```

---

## Passo 3 — Dependent items (JSONPath)

Master = coletor correspondente. `value_type` e `units` conforme abaixo.

**Sistema** (master = Performance):
| Item | JSONPath | Units |
|---|---|---|
| System: Read IOPS | `$.system.vdisk_r_io` | (vazio) |
| System: Write IOPS | `$.system.vdisk_w_io` | |
| System: Read throughput | `$.system.vdisk_r_mb` | `MBps` |
| System: Write throughput | `$.system.vdisk_w_mb` | `MBps` |
| System: Read latency | `$.system.vdisk_r_ms` | `ms` |
| System: Write latency | `$.system.vdisk_w_ms` | `ms` |
| System: CPU utilization | `$.system.cpu_pc` | `%` |
| System: Cache utilization | `$.system.total_cache_pc` | `%` |

> ⚠️ Os `stat_name` exatos (`vdisk_r_io`, `vdisk_r_mb`, `vdisk_r_ms`, `cpu_pc`,
> `total_cache_pc`, …) **devem ser confirmados** rodando `lssystemstats` no array real
> (ver Verificação).

**Saúde** (master = Capacity): `Health: Unfixed events` ← `$.unfixed_events`.

---

## Passo 4 — Discovery rules (LLD) + prototypes

Todas do tipo **DEPENDENT**, penduradas no coletor master, extraindo macros via
`lld_macro_paths`.

- **Nodes** (master = Performance): `$.nodes` →
  `{#NODE_ID}`=`$.node_id`, `{#NODE_NAME}`=`$.node_name`.
  Item prototypes (dependent, JSONPath com filtro):
  `$.nodes[?(@.node_id=='{#NODE_ID}')].stats.vdisk_r_io.first()` (e análogos w_io, r_mb,
  w_mb). Nomes: `Node [{#NODE_NAME}]: ...`.

- **Volumes** (master = Capacity): `$.volumes` →
  `{#VOLUME_ID}`=`$.id`, `{#VOLUME_NAME}`=`$.name`, `{#POOL_NAME}`=`$.pool`.
  Prototypes: `Space utilization` ← `$.volumes[?(@.id=='{#VOLUME_ID}')].pct.first()` (`%`);
  `Capacity` ← `.capacity` (`B`); `Used space` ← `.used` (`B`).
  **Trigger prototypes:**
  - `>= {$VOLUME_PCT_CRIT}` → HIGH — "Volume [{#VOLUME_NAME}]: consumo crítico"
  - `>= {$VOLUME_PCT_WARN}` → WARNING — "Volume [{#VOLUME_NAME}]: consumo alto"

- **Pools** (master = Capacity): `$.pools` →
  `{#POOL_ID}`=`$.id`, `{#POOL_NAME}`=`$.name`.
  Prototypes: `Space utilization` (`%`), `Capacity` (`B`), `Free space` (`B`), `Status`.
  Trigger prototypes: `>= {$POOL_PCT_CRIT}` → HIGH; `>= {$POOL_PCT_WARN}` → WARNING.

---

## Passo 5 — SNMP Traps (via MIB)

MIB raiz `ibm2145TSVE = .1.3.6.1.4.1.2.6.190`. Requer **interface SNMP** no host.
Três itens `snmptrap[...]` casando por OID; preprocessing (regex sobre o texto do trap)
extrai varbinds úteis para a mensagem.

| Item | key (regex do OID) | Trigger |
|---|---|---|
| Event: Error trap | `snmptrap[".1.3.6.1.4.1.2.6.190.1"]` | HIGH, `manual_close: YES` |
| Event: Warning trap | `snmptrap[".1.3.6.1.4.1.2.6.190.2"]` | WARNING, `manual_close: YES` |
| Event: Informational trap | `snmptrap[".1.3.6.1.4.1.2.6.190.3"]` | INFO/log |

Varbinds na mensagem (OIDs `...190.4.N`): ERRC `.4`, OBJT `.11`, OBJI `.12`, OBJN `.17`,
NODE `.8`, ERRS `.9`, TIME `.10`, CLUS `.7`. Preprocessing "Regular expression" para
capturar código do erro e nome do objeto e compor `event.opdata`.

---

## Passo 6 — Triggers de infraestrutura

- **Coletor indisponível:** `nodata()` sobre cada coletor (Performance/Capacity) →
  WARNING "coleta REST parada" (evita mascarar falhas).
- **Availability: ICMP ping** (item SIMPLE `icmpping`) + trigger `max(...,#3)=0` → HIGH
  "Ping Down".
- **Health: Unfixed events** `> 0` → WARNING "há eventos não resolvidos no array"
  (rede de segurança dos traps).

---

## Passo 7 — README novo

Documentar (infra fora do template):
1. **REST API:** liberar porta **7443** do Zabbix (server/proxy) → storage; criar usuário
   de monitoramento; preencher macros `{$API_*}`, `{$IBM.CERT.VERIFY}`.
2. **SNMP traps:** no Zabbix, `snmptrapd` + `SNMPTrapperFile`; importar `SVC_MIB_9.1.0.MIB`
   no snmptrapd (resolução de nomes, opcional); no array `mksnmpserver -ip <zabbix>
   -community <community>`; interface SNMP no host com `{$SNMP_COMMUNITY}`.
3. **Import:** importar `ibm_flashsystem_template.yaml`; vincular ao host; setar macros.
4. Tabela de macros e intervalos parametrizáveis.

---

## Verificação (end-to-end, contra o array 8.5 real)

1. **Import:** importar o YAML no Zabbix 6.4 sem erros de schema (`yamllint` antes ajuda).
2. **Auth + performance:** validar `POST /rest/auth` retorna token; conferir os `stat_name`
   reais de `lssystemstats`/`lsnodestats` e ajustar os JSONPaths (Passo 3/4). Test item no
   Zabbix ("Test" → "Get value").
3. **Capacidade:** confirmar campos de `lsvdisk`/`lsmdiskgrp` (`capacity`, `used_capacity`,
   `free_capacity`, `mdisk_grp_name`, `status`) e se o flag `bytes` é aceito; `toBytes`
   cobre ambos, mas confirmar unidade final = bytes.
4. **LLD:** confirmar que volumes/nós/pools são descobertos e prototypes populam.
5. **Thresholds:** forçar/observar disparo dos triggers de % (volume e pool).
6. **Traps:** gerar evento de teste no array; confirmar recepção no `snmptrapd`,
   matching do item e classificação por severidade.
7. **SSL:** confirmar o comportamento do `HttpRequest` (Script item) com cert self-signed;
   se o server verificar, importar a CA do array no trust do Zabbix server.

## Pontos a confirmar durante o dev (não bloqueiam o plano)

- Nomes exatos dos `stat_name` de `lssystemstats`/`lsnodestats` (8.5).
- Encoding do flag `bytes`/`filtervalue` no corpo JSON da REST API.
- Verificação SSL no `HttpRequest` de Script item (há ou não toggle na versão).
- Traps SNMPv1 vs v2c emitidos pelo array (afeta o texto casado pelo `snmptrap[]`).

## Fora de escopo

- Performance ao vivo **por-LUN** (IOPS/MBps por volume) — exige coletor de arquivos XML de
  stats via scp. Possível fase futura.
