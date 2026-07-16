# UnoArena — Decisiones clave: alternativas, justificación y fuentes

> Cada sección sigue el formato **problema → alternativas evaluadas → decisión → justificación**,
> con citas al repositorio fuente (`../uno-system-spec/`). Las secciones 1–7 son las decisiones
> priorizadas para la defensa; las 8+ son material de respaldo para preguntas.

---

## 1. Fairness entre regiones: carreras resueltas por timestamp efectivo ajustado por RTT

**La decisión que vino de los shooters online** (lag compensation): el repo cita explícitamente
que es "the same assumption made by CS2 lag compensation" —
`../uno-system-spec/specs/ASSUMPTIONS.md` §2.5.

**Problema.** La plataforma junta jugadores de 11 regiones. En las dos acciones
first-come-first-served donde varios jugadores son elegibles a la vez — `JumpIn` y
`ChallengeUno` — el orden puro de llegada de paquetes le da ventaja estructural al jugador más
cercano al servidor (`specs/ASSUMPTIONS.md` §2.1, `specs/CONSTRAINTS.md` §9).

**Alternativas implícitas descartadas:**
- *Orden de llegada puro*: injusto en salas cross-región (el problema mismo).
- *Timestamp reportado por el cliente*: descartado de plano — regla general del sistema: "client
  clocks are not trusted" (`specs/ASSUMPTIONS.md` §1, Clock skew).

**Decisión** (`specs/ASSUMPTIONS.md` §2.2–2.4; `specs/CONSTRAINTS.md` §9):
1. El **servidor** mide el RTT de cada jugador con heartbeats periódicos; promedio rodante en
   `PlayerSession.latency_profile` (value object `LatencyProfile` —
   `design/DOMAIN_MODEL.md`, tabla de VOs). El cliente no reporta nada.
2. Para cada submission en una carrera:
   `effective_submission_time = server_arrival_time − RTT/2` (asume ruteo simétrico).
3. **Ventana de carrera de 150 ms** de llegada al servidor: submissions dentro de la ventana
   compiten; fuera de ella se rechazan con conflicto (no son "simultáneas").
4. Gana el effective time más temprano **si supera a los demás por más de ±20 ms**; si los dos
   primeros caen dentro de ±20 ms (incertidumbre de medición) **o algún participante no tiene
   RTT medido** (`measurement_available: false`): **RNG server-side** con probabilidad uniforme.
5. Antes de aplicar cualquier efecto se agrega al log inmutable un evento **`RaceResolved`** con
   todas las submissions, arrival times, effective times, RTTs usados, ganador y
   `resolution_method: EffectiveTimestamp | RNG` (`design/COMMANDS_EVENTS.md` §`RaceResolved`).

**Justificación de los números** (`specs/ASSUMPTIONS.md` §2.5): 150 ms cubre la peor diferencia
intercontinental (~300 ms RTT → ~150 ms one-way); ±20 ms es la estimación conservadora del error
de medición + asimetría de ruteo; RTT/2 es la misma aproximación aceptada en la industria
(lag compensation de CS2).

**Privacidad:** los RTT individuales **no** se exponen a otros jugadores ni espectadores; el
broadcast lleva solo ganador y método; el detalle completo queda en el log post-partida para
auditoría/disputas (`specs/ASSUMPTIONS.md` §2.6, `design/COMMANDS_EVENTS.md` §`RaceResolved`).

**Casos trabajados:** `design/FAILURE_PATHS.md` §1 (dos jump-ins simultáneos resueltos por
EffectiveTimestamp; challenge de Uno empatado resuelto por RNG).

---

## 2. Criterios de desempate en torneos

Fuentes: `../uno-system-spec/specs/TOURNAMENT_RULES.md` §3–4 y la decisión resuelta **R3** en
`specs/ASSUMPTIONS.md` §8 (donde consta que la alternativa evaluada fue "game wins vs. cumulative
points" y se eligió match wins primero).

**Contexto.** Cada sala de torneo juega un match **Bo3**; avanzan los **3 mejores** del ranking
final de la sala. Hace falta un orden total entre 10 jugadores donde la mayoría no ganó ningún
juego.

**Cascada de desempate** (`TOURNAMENT_RULES.md` §4):
1. **Más match wins** en el Bo3.
2. Empate → **menor carga acumulada de puntos de cartas** en los juegos del match (el score −22
   contribuye 22; los placed contribuyen 0).
3. Empate → **menor tiempo de finalización acumulado**: suma de `finish_timestamp` por juego
   (reloj **monotónico server-side** desde el inicio del juego — `specs/ASSUMPTIONS.md` §1);
   quien no llegó al podio en un juego contribuye el valor del **timeout duro de 20 min**.
4. Empate total → **random server-side uniforme**, con la semilla derivada de metadata del match
   (`match_id`, `room_id`) y **registrada en el evento `MatchCompleted` para auditabilidad**.

**Reglas especiales que completan el cuadro:**
- Los **forfeited rankean debajo de todos los activos** sin importar sus métricas; entre sí, por
  momento del forfeit (más temprano = peor) (`TOURNAMENT_RULES.md` §3).
- **Salas parciales**: si quedan ≤3 activos, avanzan todos los activos (§4).
- Si el forfeit deja **1 activo**, gana el match incondicionalmente y no se juegan más juegos (§3).
- **Timeout de match (20 min)** durante un juego activo: ese juego se resuelve por más puntos →
  menos cartas → random; sin `finish_timestamp` real (todos reciben el sentinel del timeout) (§3.1).
- **Lone qualifier** que no entra en ninguna sala (mínimo 2 por sala): avanza automáticamente (§7).

**Por qué esta cascada:** cada criterio mide algo progresivamente más fino y siempre disponible —
resultado del match (lo que se compite), calidad de juego (qué tan cerca quedó de cerrar la mano),
velocidad (empíricamente ordenada por el reloj monotónico), y recién al final azar auditable. El
tie-break por tiempo depende de la decisión de diseño de registrar `finish_timestamp` en cada
placement (`specs/RULESET.md` §10).

---

## 3. Bounded contexts y servicios finales: por qué estos 7 (+gateway) y no otros

Fuentes: `../uno-system-spec/design/CONTEXT_MAP.md` (definición y relaciones),
`architecture/CONTEXT_VIEW.md` §2 (mapeo 1:1 contexto→servicio),
`architecture/adr/ADR-007-gateway-bff.md`, `architecture/adr/ADR-009-analytics-not-deployed.md`,
`architecture/PERSISTENCE.md` §8.

**Principio:** un contexto = un servicio deployable = un schema de PostgreSQL con rol propio y
acceso cross-schema revocado. La frontera no es solo lógica: está reforzada a nivel permisos de
base (`implementation/01-INFRA-AND-PLATFORM.md` §3).

**Por qué no unir contextos** — cada uno tiene un requisito de consistencia y un ciclo de vida de
datos distinto (tabla completa en `architecture/PERSISTENCE.md` §8):

| Separación | Argumento |
|---|---|
| Gameplay ≠ Tournament | Gameplay es write-heavy con row-lock por juego (~100K TPS de diseño, sharded ×16); Tournament es orquestación de baja frecuencia con row-lock por match. Relación de **Partnership**: Tournament crea salas vía HTTP y reacciona a `GameCompleted` (`CONTEXT_MAP.md` §3). Además el vocabulario difiere: "forfeit" en gameplay = salir del juego; en torneo = eliminación permanente (nuances de lenguaje ubicuo en `CONTEXT_MAP.md` §2.1–2.2). |
| Ranking separado | `EloRecord` es el **único** agregado autoritativo de Elo; consumidor puro de eventos con transacción atómica multi-jugador y reversión auditada (`elo_deltas` append-only). Meterlo en gameplay acoplaría el hot path a escrituras por jugador. |
| Spectator separado | Es una proyección read-only con **ACL de privacidad** (whitelist al consumir) y un perfil de escala propio (multiplicador 10:1 de espectadores, `CAPACITY_SKETCH.md` §10). La privacidad se garantiza estructuralmente: el read model no tiene columnas para manos (`PERSISTENCE.md` §5.3). |
| Identity upstream de todos | Sesión única y JWT son transversales; ningún otro contexto puede poseer la validez de sesión sin crear dependencias circulares. |
| Moderation separado | Bajo tráfico, alto privilegio, audit-log append-only con orden write-before-effect (`PERSISTENCE.md` §7.2). Observa todo, no posee estado de juego; emite comandos correctivos upstream. |
| Matchmaking **no** es un servicio propio | La cola casual (`MatchmakingQueue`) es un agregado de Room Gameplay; el pool de qualifiers de torneo vive en `TournamentRound.qualifier_pool` (`CONTEXT_MAP.md` §2.1). No hay un contexto "matchmaking" porque cada modo tiene su dueño natural. |

**Por qué no crear un servicio más (Game BFF):** ADR-007 evaluó separar un BFF de juego
(WebSockets) del gateway REST y lo rechazó: dos deployables duplican JWT, rate limiting y la
suscripción de invalidación de sesión; el registro unificado de conexiones simplifica el push de
invalidación; y a 10M de espectadores el problema de conexiones es el mismo con uno o dos
componentes (`architecture/adr/ADR-007-gateway-bff.md`).

**Por qué se bajó un servicio (Analytics), con las 3 alternativas evaluadas** (ADR-009):
1. *Implementarlo completo* (~medio día): rechazado — no ejercita ningún mecanismo nuevo (el
   burst ya lo absorben 4 consumer groups independientes) y duplica read models existentes.
2. *Versión token* (un endpoint de stats): rechazado — "an honestly absent service is better
   than a service that is present only to be counted".
3. *Plegar sus read models en ranking*: rechazado — pondría una superficie de consulta sin
   autoridad dentro de un contexto que posee un agregado autoritativo; exactamente la erosión de
   frontera que el diseño evita. Si Analytics vuelve, vuelve como contexto propio.

La pérdida real (stats históricas por jugador) se declara, no se esconde; el schema `analytics`
igual se provisiona para que un futuro servicio arranque con la frontera ya aplicada
(ADR-009 §Consecuencias).

---

## 4. Outbox transaccional: escribir en la DB + publicar a Kafka

Fuente principal: `../uno-system-spec/architecture/adr/ADR-003-log-before-broadcast.md`.
Implementación real: `../uno-system-spec/libs/unoarena_common/unoarena_common/outbox.py`.

**Problema.** Invariante no negociable del diseño: todo cambio de estado se escribe en el log
inmutable **antes** de que cualquier cliente o servicio downstream lo observe, y los eventos
deben llegar a Kafka at-least-once. No existe operación atómica que abarque PostgreSQL y Kafka.

**Alternativas evaluadas (esto es lo importante para la defensa):**

| Opción | Por qué se rechazó |
|---|---|
| **A — Dual-write directo** (handler escribe DB y luego publica a Kafka) | El problema del dual-write es irresoluble sin transacción distribuida: crash entre commit y publish = evento en el log pero no en Kafka; orden inverso = broadcast falso de algo que no se comiteó. Una transacción distribuida (2PC) acoplaría gameplay al protocolo transaccional de Kafka con complejidad y latencia significativas (ADR-003, Opción A). |
| **B — Event sourcing completo** (solo event store; estado rehidratado por replay) | Log-before-broadcast sale gratis, pero rehidratar `GameSession` (mazo de 108 cartas, 10 jugadores, estado de turno) en cada comando implica leer 400–1000 eventos por partida; exige snapshots y un event store con su propia complejidad operativa (evolución de schema, replay en migraciones). El costo de rehidratación pega directo en la latencia por comando (ADR-003, Opción B). |
| **C — Outbox transaccional** ✅ | Una sola transacción PG escribe: estado del agregado (`game_sessions`, JSONB → procesamiento O(1)), log inmutable (`game_events`) y `outbox`. Un relay en background publica a Kafka con producer idempotente (`enable.idempotence=true`, `acks=all`) y marca `delivered`. Si la transacción comitea, log y outbox existen juntos; si aborta, ninguno. |

**Detalles de la implementación real** (`libs/unoarena_common/outbox.py`):
- Poll con `SELECT … WHERE delivered=false ORDER BY id LIMIT n FOR UPDATE SKIP LOCKED` — varios
  relays no se pisan.
- El `traceparent` W3C y el `correlation_id` del comando originante se estampan **en la fila de
  outbox dentro de la misma transacción**; el relay los eleva al header Kafka al publicar, de modo
  que el consumidor continúa la traza del comando y no la del loop del relay
  (`implementation/FINAL_DELIVERY.md` §2.2, tracing).
- Hook de "stuck row" (>15 min) para el camino DLQ.
- **El relay corre embebido en cada servicio dueño del outbox**, no como deployable aparte
  ("which is where the transactional boundary has to live" — `FINAL_DELIVERY.md` §2.3). Cada
  contexto productor tiene su outbox: `outbox` (gameplay), `tournament_outbox` + `kickoff_outbox`,
  `identity_outbox`, `ranking_outbox`, `moderation_outbox` (`architecture/PERSISTENCE.md`).

**Trade-offs aceptados** (ADR-003 §Consecuencias): 10–100 ms extra entre COMMIT y Kafka (el
jugador activo no lo sufre: su `200 OK` sale tras el commit); lag del relay monitoreado
(`outbox_undelivered_age_seconds`, alerta >30 s); consumidores deben ser idempotentes — claves de
dedup por evento en `design/CONSISTENCY_RECOVERY.md` §2.2 (p. ej. Elo casual dedup por `game_id`
vía `last_casual_game_id`).

---

## 5. Tópicos Kafka: cantidad de particiones y partition key

Fuentes: `../uno-system-spec/architecture/adr/ADR-002-message-broker.md` §Topic topology,
`architecture/CAPACITY_SKETCH.md` §3.2, `architecture/adr/ADR-006-tournament-surge.md` §Topic
Specification, `implementation/01-INFRA-AND-PLATFORM.md` §2 (valores realmente desplegados).

**Regla de diseño de la key:** la partition key es la **unidad de orden causal** que los
consumidores necesitan. El tipo de evento va como **header** (`event-type`), no en la key ni en
tópicos por tipo — menos tópicos, mismos aislamientos (ADR-002 §Decision).

| Tópico | Key | Particiones diseño / dev | Por qué esa key |
|---|---|---|---|
| `game-events` | `game_id` | **100+ / 8** | Orden total **por juego** para todos los consumidores (spectator debe ver los eventos en orden de commit; tournament necesita consistencia del match). Es el tópico dominante (~300K ev/s pico), así que se lleva la mayor cuenta de particiones para escalar consumer groups (ADR-002; CAPACITY_SKETCH §3.2). |
| `tournament-events` | `tournament_id` | 20+ / 3 | Orden por torneo (bracket, rondas). Volumen bajo (~1K ev/s en surge). |
| `tournament-kickoff` | `room_id` | **≥100 / 8** | Fan-out del surge: 100K creaciones de sala repartidas entre los pods consumidores de room-gameplay; con 100 particiones y 50 pods, ~5.000 salas/s de capacidad de consumo contra un producer rate-limited a 1.000/s (ADR-006). |
| `tournament-kickoff-dlq` | `room_id` | — / 1 | Salas que fallan tras 3 reintentos. |
| `identity-events` | `player_id` | 20+ / 3 | Orden por jugador (SessionInvalidated → ReconnectionWindowExpired debe llegar en orden a gameplay). |
| `ranking-events` | `player_id` | 20+ / 3 | Orden de EloUpdated por jugador para el leaderboard. |
| `moderation-events` | `player_id` o `game_id` | 10+ / 3 | Según el sujeto del evento (void por juego, escalada por jugador). |

**Dimensionamiento** (`CAPACITY_SKETCH.md` §3.2): con `game-events` a ~300K ev/s pico +
`ranking-events` ~100K en burst de fin de ronda, un cluster de 5 brokers da holgura (100
particiones × RF 3 = 300 réplicas, 60 por broker — ADR-002 §Consecuencias). Los valores dev (8/3)
son el scale-down documentado: misma semántica, un solo broker Redpanda
(`implementation/01` §2: "Dev partition counts are scaled down (semantics unchanged)").

**Notas defendibles:** `cleanup.policy=delete` (nunca compact — el log de juego es un stream, la
compactación perdería eventos del mismo `game_id`); retención 7 días game-events/kickoff, 30 días
el resto; offsets comiteados **después** de procesar (at-least-once) (ADR-002 §Consecuencias).

---

## 6. Redis: un rol distinto por contexto de uso

No hay "un Redis": hay **tiers con políticas de eviction opuestas**, y en el diseño a escala son
instancias físicas separadas (Redis Cluster no soporta múltiples DBs lógicas — nota en
`architecture/CONTAINER_VIEW.md` §4). En el despliegue real quedan dos:
`redis-timer` y `redis-general` (`implementation/01-INFRA-AND-PLATFORM.md` §4,
`implementation/FINAL_DELIVERY.md` §1.4: "Not interchangeable").

| Uso | Mecanismo Redis | Dueño | Fuente |
|---|---|---|---|
| **Timers de dominio** (turno 45 s, challenge 5 s, reconexión 60 s, match timeout 20 min, lobby) | String con TTL + **keyspace notifications** + token fence UUID + Lua conditional-DEL; `noeviction` obligatorio | gameplay / identity / tournament | ADR-004 (alternativas abajo) |
| **Invalidación de sesión push** | **Pub/Sub** `session:invalidated:<player_id>` → todos los gateways cierran el WS viejo | identity → gateway | ADR-005 |
| **Fan-out a espectadores** | **Streams** `spectator:stream:<game_id>` (XADD, MAXLEN ~200, XREAD BLOCK; catch-up por stream ID) | spectator | ADR-008 |
| **Cache de sesión** | Cache-aside `identity:vsf:<player_id>` TTL 60 s ± jitter (validación JWT sin llamar a identity) | gateway | `PERSISTENCE.md` §3.2 |
| **Idempotencia** | `SET NX PX` con resultado cacheado (`gameplay:idem:*`, `identity:idem:*`) | gameplay / identity | `PERSISTENCE.md` §1.2, §3.2 |
| **Rate limiting** | `INCR` fixed-window (por IP) + **ZSET sliding-window** (por usuario) | gateway | `INTEGRATION_VIEW.md` §2.1 |
| **Leaderboards** | **Sorted Sets** `ranking:leaderboard:casual|tournament` (`ZADD` post-commit, `ZRANGE REV`); `noeviction` | ranking | `PERSISTENCE.md` §4.3 |
| **Locks distribuidos** | `SETNX` con TTL corto (`gameplay:lobby-lock`, `tournament:match-lock`) | gameplay / tournament | `PERSISTENCE.md` §1.2, §2.2 |
| **Contadores AFK** | Hash `gameplay:afk:<game_id>` | gameplay | `PERSISTENCE.md` §1.2 |
| **Read models calientes de espectador** | Hashes `spectator:gameview/roomlist` (snapshot para reconexión) | spectator | `PERSISTENCE.md` §5.1 |

**Timers — alternativas evaluadas (ADR-004):**

| Opción | Rechazo |
|---|---|
| Timers in-process | No sobreviven un crash → juego colgado para siempre. |
| Tabla PG + polling | Polling a 1 s mete hasta 1 s de jitter en una ventana de 5 s (20%); a 100 ms machaca la DB con 100K juegos. |
| Mensajes Kafka diferidos | Kafka no tiene delay nativo; cancelar es complejo; +100–500 ms para timers de 5 s. |
| **Redis TTL + keyspace notifications** ✅ | Precisión de ms, crash-safe (el timer vive en Redis), cancelación O(1) atómica vía Lua, cero infraestructura nueva. |

El punto fino defendible: las notificaciones son **at-most-once**, y la compensación son dos
sweeps de reconciliación — turno cada 60 s y **challenge cada 2 s** (porque una ventana de 5 s
perdida no puede esperar 60 s) — más el token fence que hace idempotente la doble entrega
(ADR-004 §Consecuencias). El "principio general": Redis nunca es fuente de verdad — la DB sí;
Redis es fast-path y el sistema tiene siempre un fallback durable (`PERSISTENCE.md` §Guiding
Principles; ADR-005 §Rationale).

**Invalidación de sesión — alternativas (ADR-005):** Kafka broadcast (latencia 50–300 ms y
topología de consumer-group-por-instancia horrible para gateways autoscalados) y llamada
directa gRPC/HTTP con registro de conexiones (acoplamiento a direcciones internas del gateway)
fueron rechazadas; Pub/Sub fire-and-forget es aceptable porque la DB es la autoridad y el
fallback (JWT rechazado en el próximo comando + heartbeat de push saliente a 30 s) acota la
exposición.

**Espectadores: Streams vs Pub/Sub (ADR-008):** Pub/Sub no tiene historia → un micro-corte de
200 ms pierde eventos y el re-snapshot tiene race window; con 100K canales el PSUBSCRIBE escala
mal. Streams dan catch-up por última ID sin race, fan-out ilimitado y backpressure natural, al
costo de ~6 GB de memoria a 100K juegos (MAXLEN ~200, EXPIRE 24 h post-juego) — trade-off
aceptado y dimensionado (`CAPACITY_SKETCH.md` §6).

---

## 7. Métricas de negocio: cuáles y por qué

Fuentes: implementación real en
`../uno-system-spec/libs/unoarena_common/unoarena_common/metrics.py` (el catálogo que corre),
`implementation/FINAL_DELIVERY.md` §2.2, diseño más amplio en `architecture/OBSERVABILITY.md` §4.

**Criterio de selección** (docstring de `metrics.py`): responder preguntas sobre **el juego**, no
sobre el proceso — "what a tournament operator would actually watch". Se pedían ≥3; se entregan 6.
Cada métrica se **incrementa en el servicio que posee el hecho** (aunque se definen en un solo
archivo para consistencia de nombres).

| Métrica | Tipo | Por qué se eligió |
|---|---|---|
| `uno_game_events_total{event_type}` | Counter | Enganchada **al append del log inmutable**: un solo hook instrumenta todo el dominio (CardPlayed, ChallengeWindowOpened, GameCompleted, PlayerForfeited…). |
| `uno_games_active{game_type}` | Gauge | Leída de Postgres cada 2 s en vez de incrementada por transición → **se auto-repara** tras un restart en vez de derivar para siempre. |
| `uno_matchmaking_queue_depth{region}` | Gauge | "El número más legible de la demo": sube al encolar y cae a cero al formarse la sala. |
| `uno_elo_updates_total{kind}` | Counter | Separa los 3 caminos que el diseño mantiene distintos: casual, torneo y **reversión por void de moderación**. |
| `uno_websocket_connections_active{role}` | Gauge | Jugador vs espectador: el multiplicador 10:1 del capacity sketch, **medido**. |
| `uno_rate_limit_hits_total{limit_type}` + `uno_session_events_total{event}` | Counters | La escalera de abuso y la sesión única, visibles (un spike = abuso o un cliente confundiendo presupuesto humano con presupuesto de máquina). |

**Decisiones técnicas defendibles del propio archivo:**
- **Cardinalidad**: el label de request es el *template* de ruta, nunca el path resuelto — "a
  label per game id would mint a time series per game — the classic way to melt a Prometheus";
  los 404 colapsan en `<unmatched>` para que un scanner no pueda acuñar series.
- **Toda métrica de negocio lleva ≥1 label**: un Gauge sin labels materializa un hijo en 0 al
  importar el módulo, y los 6 servicios que no poseen el hecho exportarían basura.
- **Gauge vs Counter en vivo**: un bot juega una partida entera en ~2 s, más rápido que el
  refresh del gauge — por eso el counter de eventos complementa al gauge de juegos activos
  (nota de observabilidad en `FINAL_DELIVERY.md` §2.2; se explica en la demo,
  `devOps/k8s/DEMO_RUNBOOK.md` §4).
- Además de las 6: **RED estándar** por servicio (buckets del histograma con resolución de ms
  porque el presupuesto NFR del comando de juego es P99 < 200 ms — `architecture/NFR_MATRIX.md`).

---

## 8. Protocolo de cliente: WebSocket (ADR-001)

Alternativas: **HTTP polling** (500 ms de latencia media consume 10% de la ventana de challenge
de 5 s; 2M req/s de puro overhead a 1M jugadores; imposible cerrar sesión server-side) y **SSE**
(half-duplex → 2 conexiones por jugador y sin primitiva de cierre del canal de comandos).
**WebSocket** gana por: full-duplex en 1 conexión, **close frame server-side** (la primitiva
correcta para sesión única), push sub-ms tras el commit, y orden natural comando→respuesta→push.
Fallback elegido: heartbeat ping/pong de 30 s en vez de mantener un tier de long-poll.
Fuente: `../uno-system-spec/architecture/adr/ADR-001-client-protocol.md`; también resuelto como
R12 en `specs/ASSUMPTIONS.md` §8 (cita a Board Game Arena y plataformas de juegos de mesa).

## 9. Broker: Kafka vs RabbitMQ vs Redis Streams (ADR-002)

- **RabbitMQ**: sin fan-out nativo a consumer groups con lag independiente (colas competitivas),
  sin retención/replay tras ACK, orden por routing key solo con consumidor único, techo de
  throughput a 50K msg/s.
- **Redis Streams como backbone**: memoria acotada (50 MB/s de crecimiento), sin partition key
  routing (orden por juego exigiría 100K streams), no es un log durable estilo Kafka. Se retiene
  solo para el fan-out de espectadores (acotado por juego, TTL corto).
- **Kafka** ✅: consumer groups con offsets independientes (el aislamiento del burst es
  load-critical), orden por partición (correctness-critical para spectator/tournament), replay
  durable, producer idempotente que cierra la ventana del relay de outbox.
Fuente: `../uno-system-spec/architecture/adr/ADR-002-message-broker.md`.

## 10. Surge de kickoff de torneo: Kafka fan-out vs work queue en PG (ADR-006)

Para crear 100K salas en ≤120 s: la alternativa era una tabla `room_kickoff_queue` en PG con
workers `SKIP LOCKED` llamando HTTP a room-gameplay. Se eligió el tópico dedicado
`tournament-kickoff` porque el **consumer lag es backpressure natural y observable** (vs
rate-limiting HTTP artesanal propenso a retry storms), desacopla el rate del producer (relay
limitado a 1.000 salas/s desde `kickoff_outbox`) del de los consumidores (~5.000 salas/s con 50
pods), y los room IDs **UUID5 determinísticos** hacen idempotentes los reintentos; DLQ tras 3
intentos. El endpoint HTTP de creación de salas se conserva solo para el caso no-surge (Final
Room). Fuente: `../uno-system-spec/architecture/adr/ADR-006-tournament-surge.md`.

## 11. Concurrencia: `state_version` optimista + idempotency keys

Cada juego tiene un **state version monotónico**; todo comando lleva la última versión observada
y se rechaza con `409` + versión autoritativa + `reconcile_url` si no coincide (contrato exacto
verificado en `../uno-system-spec/implementation/e2e/01-casual-games.md` §T1.4). Cada comando
lleva además un **UUID de idempotencia**: el reintento devuelve el resultado original sin
reprocesar. La serialización física es un row-lock PG por `game_id` — no hay coordinación
distribuida. Fuentes: `specs/CONSTRAINTS.md` §9, `design/CONSISTENCY_RECOVERY.md` §2.1,
`architecture/README.md`.

## 12. Autenticación: JWT + `valid_sessions_from` (híbrido, R11)

JWT RS256 para verificación de firma **stateless** (escala horizontal, identity solo firma) + un
único registro server-side por jugador (`valid_sessions_from`): un login nuevo actualiza el
timestamp y todos los tokens anteriores quedan inválidos — sesión única sin blocklist creciente,
un registro por jugador (no por token). Cache-aside en Redis con TTL 60 s para no llamar a
identity en cada request. Fuente: `../uno-system-spec/specs/ASSUMPTIONS.md` §7,
`design/DOMAIN_MODEL.md` (orden: primero se emite el JWT nuevo, después se invalida — nunca hay
ventana sin token válido).

## 13. Persistencia: DB-per-context y sharding del gameplay

- **Sin base compartida**: cada contexto tiene necesidades de consistencia y ciclos de vida de
  datos distintos (tabla comparativa en `../uno-system-spec/architecture/PERSISTENCE.md` §8);
  una DB compartida acoplaría migraciones y habilitaría queries que saltean invariantes. En el
  perfil de curso: 1 servidor, 7 schemas, 7 roles, `REVOKE` cruzado verificado por test.
- **PostgreSQL como write-store por defecto**: ACID, row-locks, JSONB (estado del juego),
  familiaridad operativa; Redis nunca es fuente de verdad (`PERSISTENCE.md` §Guiding Principles).
- **Sharding ×16 por `game_id % 16`** para el gameplay a escala: ~100K TPS contra ~10–30K TPS por
  instancia PG en workloads con row-lock; 16 shards ≈ 6.250 TPS c/u con headroom; cada pod
  conecta a los 16 shards y rutea por módulo; camino de migración 4→8→16
  (`architecture/CAPACITY_SKETCH.md` §5).

## 14. Diseño del Elo (por si preguntan fórmulas)

- Extensión multi-jugador por placement: cada juego de N jugadores se trata como N−1 duelos
  virtuales; `S_i = (N − rank_i)/(N − 1)`, esperado pairwise clásico, K decae 32→16→12 con la
  experiencia; **bono +3** si el déficit de cartas es ≤80% del promedio de la sala
  (`specs/CONSTRAINTS.md` §5.2; ejemplo numérico trabajado y verificado por golden test
  `test_worked_example_corrected` en `services/ranking-service/elo.py` —
  `specs/ASSUMPTIONS.md` §4.1).
- Torneo: **una sola actualización post-torneo** (evita distorsiones mid-tournament y usa el
  placement final completo), K=40 (mayor peso), buckets por ronda de eliminación sub-ordenados
  por win rate (`specs/TOURNAMENT_RULES.md` §8, `specs/ASSUMPTIONS.md` §4.2).
- Forfeit = último lugar; juegos voided/cancelados = Elo revertido con deltas negativos
  append-only, nunca se borra historia (`architecture/PERSISTENCE.md` §4.5).

## 15. Decisiones DevOps (checkpoint anterior — respaldo)

Fuente: `../uno-system-spec/devOps/adr/ADR-001…005`.

| ADR | Decisión | Alternativa rechazada |
|---|---|---|
| ADR-001 | Helm desde el pipeline (push) con `rollout status` como gate | GitOps (Argo/Flux): señal de readiness asíncrona que oscurece la traza `deploy → smoke` en un solo pipeline |
| ADR-002 | `rules:changes` por servicio en el monorepo | Rebuild total por push (viola el guardrail); polyrepo (fuera de alcance) |
| ADR-003 | **Build once, promote by digest** (dotenv artifact con `IMAGE_DIGEST`, prod manual + tag-gated) | Rebuild por ambiente (drift); tags mutables (`:staging` sobreescribible, no auditable) |
| ADR-004 | Variables GitLab enmascaradas + K8s Secrets/imagePullSecret | Vault/ESO (platform engineering fuera de alcance); secrets cifrados en repo (SOPS) |
| ADR-005 | Pact consumer-driven en stage `test`, bloquea build de **ambos** lados | Lint de schema solamente (no captura expectativas de comportamiento); stage tardío (desperdicia builds) |

## 16. Otras decisiones citables rápidas

- **Réplica única del api-gateway** y su remediación Redis Pub/Sub: decisión consciente
  documentada en `implementation/FINAL_DELIVERY.md` §1.5 y en los values del chart.
- **Sticky routing por juego** en el gateway para que el fan-out in-memory alcance a todos los
  jugadores de la sala sin coordinación entre instancias (`architecture/INTEGRATION_VIEW.md` §1.2).
- **Throttling adaptativo** del gateway ante presión de gameplay (P99 >200 ms → recorta cupo pero
  **preserva `CallUno` y challenges** porque la ventana de 5 s es correctness-critical)
  (`architecture/INTEGRATION_VIEW.md` §2.2).
- **Redpanda en dev** por simplicidad (1 contenedor, protocolo Kafka byte-compatible; si staging
  corre Kafka real, el código no cambia) (`implementation/01-INFRA-AND-PLATFORM.md` §1 — sección
  "Why Redpanda").
- **Sin instrumentar los WebSockets en tracing**: un socket vive toda la partida → una traza de
  201 spans ilegible; excluido a propósito, cada comando es su propia traza
  (`implementation/FINAL_DELIVERY.md` §2.2).
- **Umbrales de arranque de ronda** (formar salas con 1%/10%/50%/100% de qualifiers según ronda)
  para balancear aleatoriedad del reshuffle vs espera (`specs/TOURNAMENT_RULES.md` §5).
