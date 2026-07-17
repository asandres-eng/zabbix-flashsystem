# Design: Monitoramento IBM FlashSystem via REST API + SNMP Traps

**Data:** 2026-07-16
**Status:** Aprovado (aguardando revisão do spec)

## Problema

O template atual monitora apenas **status** (online/fault) e **capacidade por pool**
(MDiskGroup). Faltam:

1. Performance do storage como um todo — vazão (MB/s) e IOPS.
2. Performance por nó/canister.
3. Consumo percentual **por LUN** (volume), para acompanhar crescimento e alertar
   quando o threshold é atingido.

Além disso, a arquitetura atual (binário Go como *external script*, abrindo **uma
conexão SSH por item**) não escala: um array com muitos objetos gera centenas de
conexões SSH por ciclo de polling.

## Restrição técnica descoberta

A performance **ao vivo por LUN** (IOPS/MBps de cada volume) **não** é exposta por um
comando de CLI/REST no Spectrum Virtualize — esses stats por objeto só existem nos
arquivos XML de estatística, coletados por scp do nó. Decisão do usuário: **deixar
performance por-LUN fora do escopo por ora**. Performance de sistema e por nó
(`lssystemstats`/`lsnodestats`) atende à necessidade de vazão global.

## Decisões de escopo

- Reescrever para **REST API nativa** (HTTP), aposentando o binário Go e o SSH-por-item.
- Ambiente-alvo: FlashSystem **code level 8.5+**, com acesso a equipamento real para teste.
- Performance ao vivo por-LUN: **fora de escopo** nesta fase.
- Saúde/status de hardware e falhas lógicas: via **SNMP traps** (evento assíncrono),
  usando a `SVC_MIB_9.1.0.MIB`.
- Rede de segurança: **um** check REST leve de saúde geral (contagem de eventos não
  resolvidos), já que traps só capturam mudanças de estado.

## Arquitetura (híbrida)

### A) Métricas — REST API (polling)

A REST API do Spectrum Virtualize (porta 7443) usa token: `POST /rest/auth` com
`X-Auth-Username`/`X-Auth-Password` → token; depois `POST /rest/<comando>` com
`X-Auth-Token`. O HTTP agent do Zabbix não encadeia auth→uso nativamente, então os
coletores são **itens do tipo Script (JavaScript)** que, num único run, autenticam e
disparam os comandos do grupo, retornando JSON. Os dados individuais penduram como
**dependent items** com preprocessing JSONPath. Cert self-signed → verificação SSL
desligável por macro.

**Coletores mestres (agrupados por cadência, intervalo parametrizado):**

| Coletor | Comandos REST | Intervalo (macro / default) |
|---|---|---|
| Performance | `lssystemstats`, `lsnodestats` | `{$IBM.PERF.INTERVAL}` / **2m** |
| Capacidade  | `lsvdisk` (bytes), `lsmdiskgrp`, `lseventlog` (heartbeat) | `{$IBM.CAP.INTERVAL}` / **10m** |

**Métricas entregues:**

- **Sistema** (`lssystemstats`): IOPS leitura/escrita, MB/s leitura/escrita, latência
  r/w (ms), CPU %, cache %.
- **Por nó/canister** (`lsnodestats`, via LLD de nós): mesmos indicadores por nó.
- **Por LUN** (`lsvdisk`, via LLD de volumes): item calculado
  `used_capacity / capacity * 100` → % de consumo + crescimento por volume.
- **Por pool** (`lsmdiskgrp`, via LLD de pools): capacidade, livre, % usado, status.
- **Heartbeat de saúde** (`lseventlog -filtervalue fixed=no`): contagem de eventos não
  resolvidos, como sanidade complementar aos traps.

**Nota de honestidade (volumes fully-allocated):** em volumes *fully-allocated*,
`used_capacity == capacity` (sempre 100%); o crescimento real só é observável em
volumes *thin/compressed*. Todos são monitorados, mas o alerta de crescimento faz
sentido nos thin.

### B) Saúde/Status — SNMP Traps (via MIB)

Raiz da MIB: `ibm2145TSVE = .1.3.6.1.4.1.2.6.190`. Três NOTIFICATION-TYPE mapeiam
direto para severidade:

| Trap | OID | Severidade |
|---|---|---|
| `tsveETrap` (erro) | `.1.3.6.1.4.1.2.6.190.1` | HIGH/DISASTER |
| `tsveWTrap` (warning) | `.1.3.6.1.4.1.2.6.190.2` | WARNING |
| `tsveITrap` (informação) | `.1.3.6.1.4.1.2.6.190.3` | INFO |

Varbinds (`ibm2145TSVEObjects .4.N`) usados na mensagem/trigger:
`tsveERRC` (error code, .4), `tsveOBJT` (object type, .11), `tsveOBJI` (object ID, .12),
`tsveOBJN` (object name, .17), `tsveNODE` (.8), `tsveERRS` (sequence, .9),
`tsveTIME` (.10), `tsveCLUS` (cluster, .7), `tsveMACH`/`tsveSERI` (máquina/serial).

Isso **substitui** o polling de status de drives, enclosures, baterias, canisters,
PSUs, arrays e falha de volume — e elimina as LLDs de hardware correspondentes.

**Itens no template:** 3 itens `snmptrap[...]` casando por OID de trap, com preprocessing
extraindo os varbinds relevantes. Triggers: Erro→HIGH, Warning→WARNING, Info→log.
Como traps são eventos (SVC não envia clear confiável): `manual_close: YES` +
auto-resolve por tempo opcional.

**Requisitos de infra (documentados no README, fora do template):**
- Interface **SNMP** no host Zabbix (traps casam host por IP, depois item por regex).
- Lado Zabbix: `snmptrapd` + `SNMPTrapperFile` configurados; MIB carregada no
  `snmptrapd` para resolução de nomes (opcional).
- Lado array: `mksnmpserver -ip <zabbix> -community <community>`.
- **ICMP ping** mantido como heartbeat de disponibilidade.

## Macros

| Macro | Default | Uso |
|---|---|---|
| `{$API_USER}` | — | Usuário REST |
| `{$API_PASSWORD}` | — | Senha REST |
| `{$API_PORT}` | `7443` | Porta REST |
| `{$IBM.CERT.VERIFY}` | `0` | Verificação SSL (self-signed) |
| `{$IBM.PERF.INTERVAL}` | `2m` | Intervalo coletor de performance |
| `{$IBM.CAP.INTERVAL}` | `10m` | Intervalo coletor de capacidade |
| `{$SNMP_COMMUNITY}` | `public` | Community dos traps |
| `{$VOLUME_PCT_WARN}` | `80` | Threshold warning % por LUN |
| `{$VOLUME_PCT_CRIT}` | `90` | Threshold crítico % por LUN |
| `{$POOL_PCT_WARN}` | `80` | Threshold warning % por pool |
| `{$POOL_PCT_CRIT}` | `95` | Threshold crítico % por pool |

(`{$POOL_PCT_*}` renomeiam os antigos `{$MDISKGROUP_FREE_MIN_*}`.)

## Descoberta (LLD)

A partir dos JSONs dos coletores REST:
- **Volumes** (`{#VDISK_ID}`, `{#VDISK_NAME}`, `{#POOL_NAME}`)
- **Nós/canisters** (`{#NODE_ID}`, `{#NODE_NAME}`)
- **Pools** (`{#POOL_ID}`, `{#POOL_NAME}`)

LLDs de hardware (drives/enclosures/etc.) são removidas — cobertas por traps.

## Componentes / Unidades

1. **Coletor de Performance** (Script item JS) — auth + `lssystemstats`/`lsnodestats` → JSON.
2. **Coletor de Capacidade** (Script item JS) — auth + `lsvdisk`/`lsmdiskgrp`/`lseventlog` → JSON.
3. **Módulo de auth REST** (função JS reutilizável nos dois coletores).
4. **Dependent items + preprocessing** (JSONPath) — métricas de sistema, nó, volume, pool.
5. **LLD** — volumes, nós, pools.
6. **Itens SNMP trap** (3) + preprocessing de varbinds.
7. **Triggers** — thresholds de capacidade (macros) + severidades de trap.
8. **Interface/macros/README** — infra SNMP e credenciais REST.

## Tratamento de erros

- Falha de auth/HTTP no Script item → item unsupported; trigger de "coletor
  indisponível" (nodata) para não mascarar problemas.
- Cert self-signed → `{$IBM.CERT.VERIFY}=0` por padrão.
- Token expira por inatividade → cada run reautentica (stateless), aceitável.
- Divisão por zero em `% por LUN` (capacity=0) → preprocessing trata (retorna 0).

## Testes

Validação incremental contra o array 8.5 real:
1. Fluxo de auth + `lssystemstats` → confirmar JSON e token.
2. Cada coletor isolado → confirmar JSONPaths com a saída real antes de fechar o bloco.
3. LLD de volumes/nós/pools → confirmar macros populadas.
4. Traps: gerar um evento de teste no array (`mksnmpserver` + evento) → confirmar
   recepção e classificação por severidade no Zabbix.
5. Thresholds → validar disparo de trigger de % por LUN e por pool.

## Migração

- Remover `src/` (binário Go) e `docker-compose.yaml` do fluxo de uso.
- Reescrever `IBMStorageFS5K_template.yaml` do zero (novo modelo).
- Manter item ICMP ping.
- Atualizar README: pré-requisitos REST (porta 7443, usuário), infra SNMP trap
  (snmptrapd, mksnmpserver, MIB), e novas macros.

## Fora de escopo

- Performance ao vivo por-LUN (IOPS/MBps por volume) — exigiria coletor de arquivos
  XML de stats via scp. Possível fase futura.
