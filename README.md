# presentacion-uno-system

Presentación para la defensa de **UnoArena** — el sistema diseñado e implementado en el
repositorio hermano [`../uno-system-spec/`](../uno-system-spec/).

## Presentación

`main.tex` — slides en LaTeX/Beamer (tema metropolis, 16:9). Compilar con:

```bash
latexmk -pdf main.tex
```

Estructura: portada (Arquitectura de microservicios — Grupo 5) → sistema en números →
arquitectura final → bounded contexts → flujo de jugar una carta → decisiones (outbox, Kafka y
particiones, fairness RTT, desempates, Redis, surge de kickoff) → alcance honesto →
observabilidad → agenda de demo → gracias.

## Spec files

Contexto destilado del repositorio fuente, con citas a los documentos originales para que
cualquier modelo/persona pueda seguir y profundizar cada tema:

| Archivo | Contenido |
|---|---|
| [specs/00-contexto-general.md](specs/00-contexto-general.md) | Qué es UnoArena, mapa del repo fuente, bounded contexts, invariantes, stack, flujo del hot path, reglas de dominio esenciales, estado de entrega y gaps |
| [specs/01-decisiones-clave.md](specs/01-decisiones-clave.md) | Las decisiones a defender: problema → alternativas evaluadas → decisión → justificación (RTT/race resolution, desempates de torneo, bounded contexts, outbox, particiones Kafka, usos de Redis, métricas de negocio, y ~10 decisiones de respaldo) |
| [specs/02-demo.md](specs/02-demo.md) | Guión de demo: despliegue desde cluster vacío, partida casual 1v1 con dos terminales (todos los comandos), espectador, evidencia (psql/Grafana/Jaeger), trampas conocidas |

Todas las rutas citadas son relativas a este repo (`../uno-system-spec/...`).
