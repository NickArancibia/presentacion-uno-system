# Extras — leaderboard, bracket y moderación (API pública, vía curl)

Tokens (sin re-loguear — la sesión es única y un segundo login cerraría la sesión activa de esa
cuenta):

```bash
export UNOARENA_API_URL=http://localhost:8080

# token de un jugador ya logueado, leído de su session file
TOKEN=$(python3 -c 'import json,os;print(json.load(open(os.path.expanduser("~/.unoarena/carol.session")))["token"])')

# token del admin (cuenta `admin`, promovida a role=admin vía ADMIN_USERNAMES al registrarse)
curl -s -X POST $UNOARENA_API_URL/v1/auth/register -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"pw123456"}' > /dev/null   # 409 si ya existe: inofensivo
ADMIN_TOKEN=$(curl -s -X POST $UNOARENA_API_URL/v1/auth/login -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"pw123456"}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
```

## Leaderboards (Redis ZSET de ranking, servidos por spectator)

```bash
curl -s "$UNOARENA_API_URL/v1/spectator/leaderboard?type=casual&limit=10" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

curl -s "$UNOARENA_API_URL/v1/spectator/leaderboard?type=tournament&limit=10" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

## Bracket del torneo terminado

Con el `tournament_id` que imprimió el harness del torneo:

```bash
curl -s "$UNOARENA_API_URL/v1/spectator/brackets/<tournamentId>" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

## Moderación de punta a punta: flag → void → Elo revertido → auditoría

```bash
# la última partida completada
G=$(kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena -tA \
  -c "SELECT game_id FROM gameplay.game_sessions WHERE status='completed' ORDER BY updated_at DESC LIMIT 1;")

# 1) un jugador reporta la partida (presupuesto propio de 5/hora en el gateway)
curl -s -X POST "$UNOARENA_API_URL/v1/games/$G/flag" -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"reason":"suspected_cheating"}'

# 2) la misma acción SIN rol admin → 403 (el gateway exige role=admin en /v1/admin/*)
curl -s -X POST "$UNOARENA_API_URL/v1/admin/games/$G/void" -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"reason":"x"}'

# 3) el admin anula el resultado → GameResultVoided viaja por Kafka
curl -s -X POST "$UNOARENA_API_URL/v1/admin/games/$G/void" -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H 'Content-Type: application/json' -d '{"reason":"resultado adulterado"}'

# 4) ranking revirtió el Elo con el delta exacto (y es idempotente: repetir el void no re-revierte)
curl -s "$UNOARENA_API_URL/v1/spectator/leaderboard?type=casual&limit=10" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# 5) el log público de la partida quedó marcado como anulado
curl -s "$UNOARENA_API_URL/v1/spectator/games/$G/log" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool | head -30

# 6) auditoría: la acción quedó registrada y atribuida al admin (write-before-effect)
curl -s "$UNOARENA_API_URL/v1/admin/actions" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

## Suspensión de un jugador

```bash
P=$(kubectl exec -n unoarena postgres-0 -- psql -U postgres -d unoarena -tA \
  -c "SELECT player_id FROM identity.player_profiles WHERE username='bob';")
HASTA=$(date -u -d '+5 minutes' +%Y-%m-%dT%H:%M:%SZ)

curl -s -X POST "$UNOARENA_API_URL/v1/admin/players/$P/suspend" -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H 'Content-Type: application/json' -d "{\"suspended_until\":\"$HASTA\",\"reason\":\"demo\"}"

# el siguiente request de bob es rechazado y su WebSocket vivo se cierra (push de invalidación)
```
