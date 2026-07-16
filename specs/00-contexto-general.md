# UnoArena — Contexto general del sistema

> **Propósito de este archivo:** dar a cualquier modelo/persona que trabaje en esta presentación
> el contexto completo del sistema defendido, con punteros exactos al repositorio fuente
> (`../uno-system-spec/`) para profundizar en cualquier tema. Leer esto primero; las decisiones
> con alternativas y justificación están en [01-decisiones-clave.md](./01-decisiones-clave.md) y
> el guión de demo en [02-demo.md](./02-demo.md).

---

## 1. Qué es UnoArena

Plataforma global de UNO multijugador en tiempo real con dos modos:

- **Quick Play casual**: salas de 2–10 jugadores armadas por matchmaking (los jugadores nunca
  crean salas directamente — `../uno-system-spec/specs/CONSTRAINTS.md` §2.1).
- **Torneos de eliminación masivos**: hasta **1.000.000 de jugadores**, rondas sucesivas que
  reducen el pool ~70% por ronda (top 3 de cada sala de 10 avanzan), matches **Bo3**, Final Room
  con ≤10 jugadores (`../uno-system-spec/specs/TOURNAMENT_RULES.md`).

El desafío de escala dominante: al inicio de la ronda 1 se crean **~100.000 salas simultáneas**
(`../uno-system-spec/architecture/README.md` §System Summary). Casi todas las decisiones de
arquitectura se justifican contra ese pico.

El repositorio fuente recorre el arco completo: **diseño DDD** (EventStorming, agregados,
comandos/eventos, failure paths) → **arquitectura de solución** (vistas, ADRs, NFRs) →
**implementación real** (Python/FastAPI, motor de UNO propio, outbox transaccional sobre Kafka,
timers durables en Redis, Elo, torneos Bo3, proyección de espectador con privacidad, moderación)
→ **cliente CLI canónico** → **despliegue Kubernetes en un script** con observabilidad
(`../uno-system-spec/README.md`).

## 2. Mapa del repositorio fuente

| Ruta en `../uno-system-spec/` | Contenido |
|---|---|
| `specs/` | Reglas de dominio: `RULESET.md` (cartas/turnos), `CONSTRAINTS.md` (reglas de plataforma), `TOURNAMENT_RULES.md`, `ASSUMPTIONS.md` (supuestos + decisiones resueltas R1–R12) |
| `design/` | Entregables DDD: `GLOSSARY.md`, `CONTEXT_MAP.md`, `DOMAIN_MODEL.md`, `COMMANDS_EVENTS.md`, `EVENT_FLOWS.md`, `FAILURE_PATHS.md`, `CONSISTENCY_RECOVERY.md`, `ASCII_FLOW.md`, `REQUIREMENTS_TRACEABILITY.md` |
| `architecture/` | Vistas de arquitectura (`CONTEXT_VIEW`, `CONTAINER_VIEW`, `INTEGRATION_VIEW`, `PERSISTENCE`, `CAPACITY_SKETCH`, `NFR_MATRIX`, `THREAT_MODEL`, `OBSERVABILITY`), specs por servicio en `services/`, y **ADR-001…ADR-009** en `adr/` |
| `implementation/` | Plan de build por fases (`00`–`11`), `STATUS.md`, `FINAL_DELIVERY.md` (checklist de entrega, cableado real, gaps), runbooks E2E ejecutables en `e2e/` |
| `services/<servicio>/` | Código real de cada servicio (Python/FastAPI) |
| `libs/unoarena_common/` | Librería compartida: envelope de eventos, relay de outbox, timers Redis, JWT RS256, métricas, tracing, consumer Kafka |
| `devOps/` | CI GitLab, charts Helm, ADRs de DevOps (`adr/`), despliegue kind (`k8s/deploy-all.sh`, `k8s/DEMO_RUNBOOK.md`), compose local |
| `client/` | CLI canónico (harness de evaluación de la cátedra), `client/README.md` |

Orden de lectura recomendado por el propio repo: `../uno-system-spec/README.md` §Recommended
Reading Order.

## 3. Los 7 bounded contexts (+1 cross-cutting)

Definidos en `../uno-system-spec/design/CONTEXT_MAP.md`; mapeo contexto→servicio 1:1 en
`../uno-system-spec/architecture/CONTEXT_VIEW.md` §2.

| Contexto | Servicio | Rol / posee |
|---|---|---|
| **Room Gameplay** | `room-gameplay-service` | Núcleo upstream. Dueño de `GameSession`, `Room`, `MatchmakingQueue`; todas las reglas de UNO, `state_version`, timers de turno/challenge, log inmutable de juego, outbox |
| **Tournament Orchestration** | `tournament-service` | `Tournament`, `TournamentRound`, `Match`; ciclo de rondas, tracking Bo3, timeout de match (20 min), desempates y avance, fan-out de kickoff |
| **Identity / Session** | `identity-service` | `PlayerProfile`, `PlayerSession`; JWT RS256, `valid_sessions_from` (sesión única), ventana de reconexión (60 s) |
| **Ranking** | `ranking-service` | `EloRecord` (autoridad de Elo casual y de torneo), leaderboards Redis, reversión ante void/cancel |
| **Spectator View** | `spectator-service` | Proyección read-only con **whitelist de privacidad aplicada al consumir** — las manos jamás entran al read model; `PublicGameView`, `PublicGameLog` sellado, `BracketView` |
| **Analytics / Read Models** | *(diseñado, **no desplegado** — ADR-009)* | Stats históricas por jugador, vistas de bracket; puro lado de lectura CQRS |
| **Moderation / Admin** | `moderation-service` | Log de auditoría append-only, comandos correctivos (void, cancel, suspend/ban), escalada de abuso |
| Cross-cutting | `api-gateway` | Única entrada pública: termina TLS + WebSocket, valida JWT, rate limiting, registro de conexiones, push de invalidación de sesión |

**Deployables que corren: 7** (los 7 de arriba menos analytics; el `outbox-relay-worker` del
diseño corre **embebido** en cada servicio vía `libs/unoarena_common/outbox.py` porque ahí vive
la frontera transaccional — `../uno-system-spec/implementation/FINAL_DELIVERY.md` §0 y
`architecture/adr/ADR-009-analytics-not-deployed.md` §Consecuencias).

## 4. Invariantes arquitectónicos clave

De `../uno-system-spec/architecture/README.md` §System Summary:

1. **Log-before-broadcast**: todo cambio de estado de juego se escribe en PostgreSQL (fila de
   agregado + `game_events` + `outbox`) en **una** transacción antes de llegar a Kafka o a
   cualquier cliente (ADR-003).
2. **Single-active-session**: un login nuevo invalida la sesión anterior; el gateway cierra el
   WebSocket viejo empujado por Redis Pub/Sub (ADR-005), código de cierre `4401`.
3. **Serialización por juego**: los comandos del mismo `game_id` se serializan con row-lock de
   PostgreSQL (`SELECT … FOR UPDATE`) + **concurrencia optimista por `state_version`** — sin
   locks distribuidos (`specs/CONSTRAINTS.md` §9).
4. **Aislamiento del surge**: el pico de `GameCompleted` al final de ronda lo absorben consumer
   groups Kafka independientes; ningún consumidor lento frena a los escritores.
5. **Privacidad estructural**: el filtro whitelist del Spectator se aplica al consumir el evento;
   el read model no tiene columnas para manos (`architecture/PERSISTENCE.md` §5.3).
6. **Consistencia eventual entre contextos**: entrega at-least-once + consumidores idempotentes
   con claves de dedup documentadas por evento; **cero transacciones distribuidas**
   (`design/CONSISTENCY_RECOVERY.md`).
7. **Timers autoritativos del servidor**: turno 45 s, ventana de challenge 5 s, reconexión 60 s,
   match timeout 20 min — Redis TTL + keyspace notifications con token fence (ADR-004).

## 5. Stack e infraestructura

- **Servicios**: Python 3.12 / FastAPI (el diseño decía "JVM o Go"; la implementación real es
  Python — ver `../uno-system-spec/services/`).
- **PostgreSQL 16**: un servidor, base `unoarena`, **7 schemas con 7 roles dedicados** y acceso
  cross-schema revocado a nivel permisos de DB (`implementation/01-INFRA-AND-PLATFORM.md` §3).
  El diseño a escala completa shardea gameplay ×16 por `game_id % 16`
  (`architecture/CAPACITY_SKETCH.md` §5); en el perfil de curso es 1 nodo pero el código resuelve
  la conexión vía `shard_for(game_id)` para que el sharding sea un cambio de config.
- **Kafka**: en dev/demo es **Redpanda** (byte-compatible, un contenedor, sin ZooKeeper —
  `implementation/01` §1); 7 tópicos (detalle de particiones y keys en
  [01-decisiones-clave.md](./01-decisiones-clave.md) §5).
- **Redis ×2 instancias** en el perfil desplegado: `redis-timer` (`noeviction` +
  `notify-keyspace-events KEA`) y `redis-general` (cache, idempotencia, rate limits,
  leaderboards, pub/sub, streams de espectador) — `implementation/01` §4. El diseño a escala
  separa más instancias por tier (`architecture/CONTAINER_VIEW.md` §4).
- **Despliegue**: `./devOps/k8s/deploy-all.sh` — de cluster kind vacío a plataforma corriendo
  (~6 min frío, ~2 min con imágenes cacheadas); sin registry (side-load de imágenes), sin cloud.
- **Observabilidad**: Prometheus (RED + 6 métricas de negocio), Grafana con dashboard
  provisionado como código ("UnoArena — Live"), Jaeger con trazas que **cruzan Kafka**
  (`traceparent` estampado en la fila de outbox dentro de la misma transacción —
  `implementation/FINAL_DELIVERY.md` §2.2).

## 6. Flujo de un comando de juego (hot path)

Fuente: `../uno-system-spec/architecture/INTEGRATION_VIEW.md` §1 y §3.1–3.2.

1. El jugador tiene **un WebSocket** contra el api-gateway (ADR-001). Cada frame de comando se
   reenvía como HTTP POST a room-gameplay con `player_id` validado en headers.
2. room-gameplay: row-lock del juego → valida `state_version` + legalidad → una transacción PG
   (estado + `game_events` + `outbox`) → `COMMIT` → `200 OK` con los eventos resultantes.
3. El gateway hace push de esos eventos a los demás jugadores de la sala desde su registro de
   conexiones en memoria (sin esperar a Kafka) — latencia e2e sub-100 ms.
4. El relay de outbox publica a Kafka (`game-events`, key `game_id`, producer idempotente,
   `acks=all`); ranking, tournament, spectator y moderation consumen con groups independientes.
5. Los espectadores reciben los eventos filtrados vía Redis Streams (`XREAD BLOCK`), con catch-up
   por stream ID al reconectar (ADR-008).

## 7. Reglas de dominio esenciales (para responder preguntas en la defensa)

- **Turnos**: timer de 45 s; al expirar para un jugador conectado, el server ejecuta "robar 1 y
  pasar"; 3 expiraciones consecutivas = forfeit por AFK (`specs/CONSTRAINTS.md` §2.3–2.4).
- **Desconexión**: ventana de reconexión de 60 s (turnos salteados mientras tanto, sin acumular
  AFK); expiración = forfeit permanente (`specs/CONSTRAINTS.md` §2.5).
- **Fin de juego**: el 1° en vaciar la mano gana; el juego sigue hasta determinar 2° y 3°
  (con `finish_timestamp` de reloj monotónico server-side); termina con 3 placements o 1 activo
  (`specs/RULESET.md` §10).
- **Elo**: dos ratings separados — casual (por juego, extensión multi-jugador por placement,
  K 32/16/12 por experiencia, bono +3 por performance) y torneo (una sola vez post-torneo, K=40)
  (`specs/CONSTRAINTS.md` §5, `specs/TOURNAMENT_RULES.md` §8, `specs/ASSUMPTIONS.md` §4).
  Forfeit = último puesto, siempre.
- **Wild Draw Four challenge**: verificación de mano **solo server-side**; la mano nunca se
  revela durante el juego y aparece en el log público post-partida (`specs/CONSTRAINTS.md` §4.2).
- **Log de juego**: inmutable, público recién al completarse la partida; cualquiera puede
  flaggear un juego para revisión admin (`specs/CONSTRAINTS.md` §6).
- **Regiones**: 11 regiones definidas, matchmaking por proximidad regional primero y Elo después,
  con grupos de adyacencia (`specs/CONSTRAINTS.md` §8).
- **Rate limiting**: multi-capa (IP, usuario, acción, flag, admin) con escalada de abuso
  5 violaciones→warning, 3 warnings→suspensión 15 min (`specs/CONSTRAINTS.md` §10,
  `architecture/INTEGRATION_VIEW.md` §2).

## 8. Estado de entrega y gaps honestos

De `../uno-system-spec/implementation/FINAL_DELIVERY.md` y `README.md` §Honest gaps — decir
estos gaps **antes** de que los pregunten (así lo recomienda el propio
`devOps/k8s/DEMO_RUNBOOK.md` §Known gaps):

1. **analytics-service no desplegado** — decisión registrada en ADR-009: su mecanismo (absorber
   el burst) ya lo demuestran 4 consumer groups independientes y sus read models duplican los de
   ranking/tournament/spectator. Lo que sí se pierde: estadísticas históricas por jugador.
2. **outbox-relay-worker no desplegado como servicio** — corre embebido donde vive la frontera
   transaccional.
3. **api-gateway con 1 réplica** — su registro de WebSockets es un dict in-process; con 2
   réplicas, el `POST /internal/push/{player_id}` caería en un pod aleatorio y se perderían
   pushes. Remediación documentada: fan-out Redis Pub/Sub (mismo mecanismo que ya usa la
   invalidación de sesión) — `implementation/FINAL_DELIVERY.md` §1.5.
4. **`bot --room <id>` no cableado** (las salas las asigna matchmaking/torneo).
5. **Nodo único**: 1 Postgres (7 schemas, no ×16 shards), 1 broker Redpanda — perfil de
   scale-down documentado; mismos code paths y contratos.

Verificado E2E (2026-07-14, `FINAL_DELIVERY.md` §2.1): despliegue frío completo, partida
interactiva humano-vs-bot con espectador (cero fuga de manos), Elo por Kafka (1019/984),
torneo Bo3 de 4 bots hasta campeón con Elo de torneo, single-active-session 6/6, 7/7 targets de
Prometheus, traza Jaeger de 12 spans cruzando Kafka.

## 9. Checkpoints previos (contexto de la materia)

- **DevOps checkpoint**: monorepo GitLab, 9 servicios con `test → build → deliver` y api-gateway
  hasta `deploy-staging → integration-staging`; Helm desde pipeline, build-once/promote-by-digest,
  Pact, smoke con el CLI. ADRs propios en `../uno-system-spec/devOps/adr/` (deployment model,
  change detection por `rules:changes`, promoción por digest, secrets, contract tests).
- **Client checkpoint**: superficie canónica del CLI + contrato de salida JSON-lines + Docker
  (`../uno-system-spec/Client-Checkpoint.md`, `client/README.md`). El CLI hoy maneja el backend
  real y es el harness funcional de la defensa.
