# UnoArena — Guión de demo: partida casual 1v1 con dos terminales

> Basado en el runbook ya ensayado del repositorio fuente:
> `../uno-system-spec/devOps/k8s/DEMO_RUNBOOK.md` (guión de defensa completo, comando por
> comando), `../uno-system-spec/implementation/e2e/01-casual-games.md` §T1.2 (partida
> interactiva humano-vs-humano) y `../uno-system-spec/client/README.md` (superficie del CLI).
> **Nada se improvisa el día de la defensa**: todo lo de abajo fue ejecutado tal cual en el
> ensayo en frío del 2026-07-14 (`../uno-system-spec/implementation/FINAL_DELIVERY.md` §2.1).

---

## 0. Preparación (la mañana de la defensa, en casa)

```bash
cd ../uno-system-spec
kind --version && kubectl version --client && helm version --short && docker --version

./devOps/k8s/deploy-all.sh --destroy   # arrancar realmente desde cero
./devOps/k8s/deploy-all.sh             # ensayo frío completo (~6 min)
./devOps/k8s/deploy-all.sh --destroy   # dejar el cluster DESTRUIDO…
```

…pero con las **imágenes ya en cache de Docker**: el deploy en vivo baja de ~6 min a **~2 min**
(`DEMO_RUNBOOK.md` §0). Los examinadores quieren ver el despliegue desde cluster vacío.

## 1. Despliegue en vivo (~2 min)

```bash
./devOps/k8s/deploy-all.sh
```

Mientras corre, narrar lo que hace (§1 del runbook): crea el cluster kind → side-load de las 7
imágenes (sin registry) → chart de infra (Postgres con 7 schemas aislados, redis-timer,
redis-general, Redpanda + 7 tópicos) → Prometheus/Grafana/Jaeger → los 7 servicios.

Mostrar que es un cluster real:

```bash
kubectl get pods -n unoarena
kubectl get svc,statefulset,deploy -n unoarena
```

URLs que imprime el script (dejarlas a la vista):

| Qué | Dónde |
|---|---|
| API (target del CLI) | `http://localhost:8080` |
| Grafana — dashboard **UnoArena — Live**, sin login | `http://localhost:3000` |
| Prometheus | `http://localhost:9090` |
| Jaeger | `http://localhost:16686` |

**Abrir Grafana ANTES de jugar** (§2 del runbook): el dashboard vacío que se llena en vivo es
parte del guión.

## 2. Setup del CLI y ⚠️ la única trampa de la demo

```bash
export UNOARENA_API_URL=http://localhost:8080
cd client && pip install -r requirements.txt   # una vez
```

**Cada terminal necesita su propio session file.** El CLI guarda el token en una ruta fija
(`~/.unoarena/session.json`); dos terminales que la comparten actúan como **el mismo jugador** y
el matchmaking nunca los empareja (`DEMO_RUNBOOK.md` §3):

```bash
export UNOARENA_SESSION_FILE=~/.unoarena/alice.session   # terminal A
export UNOARENA_SESSION_FILE=~/.unoarena/bob.session     # terminal B
export UNOARENA_SESSION_FILE=~/.unoarena/carol.session   # terminal C (espectador)
```

## 3. Autenticación (y de paso, single-active-session)

Terminal A:
```bash
python -m unoarena_cli register --user alice --pass 'Passw0rd!'
python -m unoarena_cli login    --user alice --pass 'Passw0rd!'
python -m unoarena_cli whoami
```
Terminal B: ídem con `bob`. Terminal C: ídem con `carol`.

**Mini-demo de sesión única** (vale la pena, §3.1 del runbook): loguear `alice` desde otra
terminal → el WebSocket vivo de la primera se cierra con **código 4401** — es el push de
invalidación por Redis Pub/Sub (ADR-005), no un flag de base de datos. `play` reporta
`session superseded` y sale con código 3 (`client/README.md` §Session supersession).

## 4. La partida casual 1v1 — dos terminales interactivas

Ambas terminales entran a la cola; el matchmaker forma la sala de 2 y arranca:

```bash
# Terminal A (alice)
python -m unoarena_cli play --casual
# Terminal B (bob)
python -m unoarena_cli play --casual
```

`play` muestra: tablero, mano en notación canónica (`R5`, `G0`, `BSKIP`, `Y+2`, `RREV`, `WILD`,
`WILD+4`), feed de eventos en vivo empujado por el WebSocket (los movimientos del rival se
narran **sin polling**) y las cartas jugables marcadas con `*` (`client/README.md` §Interactive
play).

### Comandos interactivos a exercitar (todos)

| Comando | Qué probar en vivo |
|---|---|
| `play <n> [R\|G\|B\|Y]` | Jugar por índice; un `WILD`/`WILD+4` exige declarar color |
| `draw` | Robar; si hay cadena de +2 apilados, **paga la cadena** |
| `uno` | Cantar UNO al quedar con 1 carta (ventana de 5 s) |
| `challenge` | Desafiar la ventana abierta de Uno! o de WILD+4 |
| `state` | Redibujar el tablero |
| `forfeit` | Rendirse (mostrar solo si sobra tiempo — termina la partida) |
| `quit` | Salir SIN rendirse (la ventana de reconexión de 60 s queda corriendo) |

Momentos que conviene provocar y narrar:
- **UNO + challenge**: al quedar con 1 carta, cantar `uno` dentro de los 5 s; o dejar que el otro
  haga `challenge` para mostrar la penalización.
- **WILD+4 apilado**: pagarlo con `draw`; en el ensayo se jugó exactamente esto (11 cartas por
  índice, WILD con color, UNO, +4 apilado pagado — `FINAL_DELIVERY.md` §2.1).
- **Comando stale**: si la mesa se mueve debajo tuyo, el backend responde
  `409 stale_state_version` con la versión autoritativa; el CLI imprime el conflicto, reproduce
  los eventos perdidos del log y redibuja (`client/README.md` §Stale commands). Es la
  concurrencia optimista por `state_version` de `specs/CONSTRAINTS.md` §9, visible en vivo.
- Al final: **placements + scores** impresos en ambas terminales, `GameCompleted` en ambos feeds.

### Terminal C — el espectador que nunca ve una mano

Obtener el room id (con `room list`, o el más race-free desde la DB —
`implementation/e2e/01-casual-games.md` §T1.3):

```bash
python -m unoarena_cli room list
python -m unoarena_cli spectate <roomId>
```

Narrar: el feed muestra jugadas, robos **como conteos** y avances de turno; la whitelist de
privacidad se aplica al consumir el evento en spectator-service — las manos jamás entran a su
store ni a su stream; el CLI además filtra claves con forma de mano como defensa en profundidad
(`client/README.md` §Spectating). Verificación mecánica de cero fuga: e2e §T1.3
(`grep -Ei '"hand"|"cards":' → no leak`).

### Alternativa si un humano se traba: humano vs bot

```bash
# Terminal B en su lugar:
UNOARENA_SESSION_FILE=~/.unoarena/bob.session python -m unoarena_cli bot --casual
```
El bot juega headless (una línea JSON por acción + summary + exit code —
`client/README.md` §Output contract). Ojo: `--max-actions` default es 10; para partidas
completas usar 300+ (`e2e/01-casual-games.md` §T1.1).

## 5. Evidencia de que no es teatro (§6 del runbook)

```bash
# El log de juego inmutable, por tipo de evento
kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena \
  -c "SELECT event_type, count(*) FROM gameplay.game_events GROUP BY 1 ORDER BY 2 DESC LIMIT 8;"

# El Elo se movió vía Kafka (winner ~1019 / loser ~984 — no es zero-sum exacto por el bono +3
# de performance de CONSTRAINTS §5.2.5)
kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena \
  -c "SELECT player_id, casual_elo, casual_games_played FROM ranking.elo_records ORDER BY casual_elo DESC;"

# Aislamiento por contexto: 7 schemas, 7 roles
kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena -c "\dn"

# Los tópicos con las particiones que pide el diseño
kubectl exec -n unoarena redpanda-0 -- rpk topic list
```

**Grafana** (§4 del runbook — negocio primero, RED después): cola de matchmaking que sube y cae
a cero al formarse la sala; eventos/s por tipo; escrituras de Elo por camino; conexiones WS
jugador vs espectador. En RED, señalar el **P99** (el promedio esconde la cola que rompe el
presupuesto de 200 ms). Si `Games in progress` marca 0 con un bot: un bot juega la partida en
~2 s, más rápido que el sampling del gauge — decirlo en voz alta, el counter de al lado contó
todos los eventos (trade-off de sampling, `DEMO_RUNBOOK.md` §4).

**Jaeger** (§5 del runbook): servicio `room-gameplay-service` → abrir la traza de ~12 spans:
gateway → room-gameplay (sync) y, 210 ms después **cruzando Kafka**, `consume GameCompleted` en
ranking (la escritura de Elo), tournament, identity y spectator. Funciona porque el
`traceparent` del comando se escribe en la fila de outbox **dentro de la misma transacción** y el
relay lo eleva al header Kafka al publicar.

## 6. Opcional si sobra tiempo: torneo Bo3 con 4 bots (~2 min)

`TOURNAMENT_MIN_PLAYERS: "4"` está deliberadamente bajo en el ConfigMap de infra para que 4 bots
alcancen para un bracket Bo3 real con campeón y Elo de torneo (`DEMO_RUNBOOK.md` §3.3,
`FINAL_DELIVERY.md` §1.3). El flujo: registrar/loguear `t1..t4` (cada uno con su session file),
el admin (`admin`, promovido vía `ADMIN_USERNAMES`) crea el torneo, y cada bot corre
`bot --tournament <id>`. En el ensayo: campeón con Elo de torneo 1020, driver exit 0.

## 7. Teardown

```bash
./devOps/k8s/deploy-all.sh --destroy
```

---

## Gaps a decir ANTES de que los pregunten (§Known gaps del runbook)

1. **api-gateway con 1 réplica** — registro de WS in-process; el fix documentado es fan-out por
   Redis Pub/Sub, el mismo mecanismo que ya usa la invalidación de sesión. Deliberado.
2. **analytics-service no desplegado** (ADR-009) — sus responsabilidades de lectura las cubren
   spectator y ranking; lo que se pierde son stats históricas por jugador.
3. **outbox-relay-worker embebido** en cada servicio — ahí vive la frontera transaccional.
4. **Nodo único** — 1 Postgres (7 schemas, no ×16 shards), 1 broker: perfil de scale-down
   documentado en `implementation/01-INFRA-AND-PLATFORM.md`; mismos code paths y contratos.
5. **`bot --room <id>` no cableado**; `room join` es un *hint* al matchmaking, no una garantía
   (`client/README.md` §Room semantics).

## Problemas conocidos / trampas operativas (aprendidas en los ensayos)

- **Session file por terminal** — la trampa #1 (ver §2).
- **Reutilizar pocas cuentas en muchas partidas seguidas** concentra requests en la misma ventana
  de rate limit y la **escalera de abuso suspende cuentas en medio del juego** (¡comportamiento
  diseñado, no bug!): 35 violaciones → 15 warnings → 5 suspensiones en un ensayo real. Para
  corridas de volumen, cuentas frescas por slot (`seed`) o espaciar olas ≥60 s
  (`e2e/01-casual-games.md` §Account-reuse warning).
- El matchmaker puede juntar 6 encolados simultáneos en **una** sala de 6 en vez de 3×2 — para un
  1v1 limpio, encolar solo a los dos jugadores de la demo.
- `bot --max-actions` default 10 → subir a 300+ para partidas completas.
- Variable de shell `GID` es read-only en zsh — usar `G` para el game id (nota de
  `e2e/01-casual-games.md` §T1.4).
- Con `seed`, la clave del JSON es `pass`, no `password` (`e2e/01-casual-games.md`).
