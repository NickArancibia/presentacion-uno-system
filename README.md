# presentacion-uno-system

Presentación para la defensa de **UnoArena** — el sistema diseñado e implementado en el
repositorio hermano [`../uno-system-spec/`](../uno-system-spec/).

## Presentación

`main.tex` — slides en LaTeX/Beamer (tema metropolis, 16:9). Compilar con:

```bash
latexmk -pdf main.tex
```

## Spec files

Contexto destilado del repositorio fuente, con citas a los documentos originales:

| Archivo | Contenido |
|---|---|
| [specs/00-contexto-general.md](specs/00-contexto-general.md) | Qué es UnoArena, mapa del repo fuente, bounded contexts, invariantes, stack, flujo del hot path, reglas de dominio esenciales |
| [specs/01-decisiones-clave.md](specs/01-decisiones-clave.md) | Las decisiones a defender: problema → alternativas evaluadas → decisión → justificación |
| [specs/02-demo.md](specs/02-demo.md) | Guión de demo ampliado, con evidencia |

---

# Demo

Todos los comandos se ejecutan desde la raíz de `../uno-system-spec/`.
Requiere `docker`, `kind`, `kubectl` y `helm`.

## 1. Despliegue desde cluster vacío

```bash
./devOps/k8s/deploy-all.sh
```

Crea el cluster kind → side-load de las 7 imágenes (sin registry) → infra (PostgreSQL con
7 schemas aislados, redis-timer, redis-general, Redpanda + 7 tópicos)
→ los 7 servicios, gateway último.

Verificar el cluster:

```bash
kubectl get pods -n unoarena
kubectl get svc,statefulset,deploy -n unoarena
```

| Qué | Dónde |
|---|---|
| API (target del CLI) | `http://localhost:8080` |
| Grafana — dashboard **UnoArena — Live**, sin login | `http://localhost:3000` |
| Prometheus | `http://localhost:9090` |
| Jaeger | `http://localhost:16686` |

Abrir Grafana antes de jugar — el dashboard se llena en vivo.

## 2. Tres terminales: bob, alice y carol

Setup del CLI (una vez):

```bash
export UNOARENA_API_URL=http://localhost:8080
cd client && pip install -r requirements.txt
```

Cada terminal usa **su propio session file** (una sesión por jugador):

```bash
# Terminal A
export UNOARENA_API_URL=http://localhost:8080
export UNOARENA_SESSION_FILE=~/.unoarena/bob.session
cd client
python -m unoarena_cli register --user bob --pass 'Passw0rd!'
python -m unoarena_cli login    --user bob --pass 'Passw0rd!'
python -m unoarena_cli whoami
```

```bash
# Terminal B — ídem con alice
export UNOARENA_SESSION_FILE=~/.unoarena/alice.session
```

```bash
# Terminal C — ídem con carol (espectadora)
export UNOARENA_SESSION_FILE=~/.unoarena/carol.session
```

Sesión única (opcional): un segundo `login` del mismo usuario cierra el WebSocket de la sesión
anterior con código **4401** — push de invalidación vía Redis Pub/Sub, no un flag en la DB.

## 3. Partida casual 1v1 — bob vs alice

Ambas terminales entran a la cola de matchmaking; al formarse la sala de 2, la partida arranca:

```bash
# Terminal A y Terminal B
python -m unoarena_cli play --casual
```

`play` muestra el tablero, la mano en notación canónica (`R5`, `BSKIP`, `Y+2`, `WILD+4`), las
cartas jugables marcadas con `*` y el feed de eventos en vivo empujado por WebSocket.

| Comando | Efecto |
|---|---|
| `play <n> [R\|G\|B\|Y]` | Jugar la carta n; `WILD`/`WILD+4` exige declarar color |
| `draw` | Robar; con +2 apilados, paga la cadena completa |
| `uno` | Cantar UNO al quedar con 1 carta (ventana de 5 s) |
| `challenge` | Desafiar la ventana abierta de Uno! o de WILD+4 |
| `state` | Redibujar el tablero |

Si la mesa se movió debajo de un comando, el backend responde `409 stale_state_version` y el CLI
reconcilia solo: reproduce los eventos perdidos del log y redibuja (concurrencia optimista por
`state_version`).

Al terminar: placements + scores en ambas terminales, `GameCompleted` en ambos feeds, y la
escritura de Elo casual visible en Grafana.

Variante humano vs bot (reemplaza la terminal B):

```bash
python -m unoarena_cli bot --casual --max-actions 300
```

## 4. Espectadora en vivo — carol

Con la partida del paso 3 en curso, en la terminal C:

```bash
python -m unoarena_cli room list
python -m unoarena_cli spectate <roomId>
```

El feed muestra jugadas, avances de turno y robos como **conteos** — nunca manos: la whitelist
de privacidad se aplica al consumir cada evento en spectator-service, antes de que llegue a
cualquier store o stream.

## 5. Torneo con 20 bots — bracket Bo3 real hasta el campeón

El harness registra los jugadores, crea y arranca el torneo como admin, lanza un bot por jugador
y sigue el bracket hasta el campeón:

```bash
python3 client/loadtest/massive_tournament.py --live --players 20 --concurrency 20
```

Con 20 jugadores: 2 salas de 10 en la ronda 1 → avanzan los top 3 de cada una → 6 → 3 → campeón.
Cada match es Bo3. Al final imprime el JSON con `champion_id` y `elapsed_seconds`.

Mientras corre, en **Grafana** (`UnoArena — Live`):

- **Matchmaking queue** — sube al encolar y cae a 0 al formarse cada sala.
- **Game events / sec, por tipo** — `CardPlayed`, `ChallengeWindowOpened`, `GameCompleted`…
- **Elo writes** — camino casual vs torneo.
- **Conexiones WebSocket** — jugadores vs espectadores.
- Fila **RED** — rate, errores y latencia P99/P50 por servicio.

En **Jaeger** (`http://localhost:16686`): servicio `room-gameplay-service` → una traza de ~12
spans: gateway → room-gameplay (síncrono) y, cruzando Kafka, `consume GameCompleted` en ranking,
tournament, identity y spectator — el `traceparent` viaja en la fila del outbox dentro de la
misma transacción.

Verificación en la base de datos:

```bash
# Elo de torneo del campeón (K=40, arranca en 1000)
kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena \
  -c "SELECT player_id, tournament_elo FROM ranking.elo_records ORDER BY tournament_elo DESC LIMIT 5;"

# Estado final del torneo
kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena \
  -c "SELECT status, champion_id IS NOT NULL AS has_champion FROM tournament.tournaments ORDER BY created_at DESC LIMIT 1;"

# El log de juego inmutable, por tipo de evento
kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena \
  -c "SELECT event_type, count(*) FROM gameplay.game_events GROUP BY 1 ORDER BY 2 DESC LIMIT 8;"

# Aislamiento por contexto: 7 schemas, 7 roles
kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena -c "\dn"

# Los tópicos con sus particiones
kubectl exec -n unoarena redpanda-0 -- rpk topic list
```

## 6. Teardown

```bash
./devOps/k8s/deploy-all.sh --destroy
```
