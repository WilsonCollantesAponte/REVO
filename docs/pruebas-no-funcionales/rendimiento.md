# Pruebas de rendimiento (carga, estrés y recursos)

Estado: **plan** (sin ejecuciones todavía).

Este documento mide **cuántos alumnos simultáneos** soporta REVO y qué recursos
consume. El tiempo de cada petición con un solo usuario está en
[`tiempo-de-respuesta.md`](tiempo-de-respuesta.md).

## Alcance

Arquitectura bajo prueba (leída del código):

| Componente | Archivo | Datos relevantes para el rendimiento |
|---|---|---|
| Pasarela nginx | `infraestructura/pasarela/nginx.conf.template` | `worker_processes auto`, 4096 conexiones por worker; `limit_req` por IP: zona `general` 30 r/s (ráfaga 60) y `acceso` 10 r/s (ráfaga 100) para login/registro; `limit_conn` 100 conexiones por IP; `proxy_read_timeout 35s`; resuelve DNS por petición (sin keepalive hacia los servicios) |
| auth / survey / ml (FastAPI + uvicorn) | `docker-compose.yml`, `services/*/Dockerfile` | `WORKERS` por defecto 2 en producción y 1 en `docker-compose.desarrollo.yml` |
| Pool de conexiones a PostgreSQL | `services/comun/revo_comun/ajustes.py`, `services/comun/revo_comun/servicio.py` | `DB_POOL_SIZE=5`, `DB_MAX_OVERFLOW=5` por proceso; al arrancar se comprueba `servicios × workers × (pool + overflow)` contra `max_connections` (reserva 15) |
| Cupos de la aplicación (Redis) | `services/comun/revo_comun/limites/politicas.py`, `services/*/politicas.py` | `read` 240/min, `admin` 120/min, `global` 1200/min por IP; `answers` 120/min y `submit_phase` 20/5 min por alumno; `predict` 20/5 min por alumno; `login` 8/15 min por cuenta; `register` 80/10 min por IP |
| Redis | `docker-compose.yml` | `maxmemory 256mb`, `allkeys-lru` |
| PostgreSQL 16 + pgvector | `infraestructura/postgres/Dockerfile`, `docker-compose.yml` | RLS en cada consulta |
| Predicción | `services/ml-service/routers/predict.py`, `services/ml-service/model/` | Regresión logística (scikit-learn); la inferencia es la ruta con más CPU según `services/ml-service/politicas.py` |
| Despliegue alternativo | `render.yaml` | Cuatro servicios en `plan: free` (se duermen; arranque en frío) |

**Importante para diseñar la carga:** los límites por IP de nginx y de la
aplicación hacen que cientos de usuarios virtuales desde **una sola IP** midan
el limitador y no el sistema. Los scripts del equipo ya lo resuelven con un
contenedor por aula (`infraestructura/carga/aula.py`,
`infraestructura/carga/ejecutar_prueba.py`, red `10.77.0.0/16`). Cualquier
herramienta nueva (k6, Locust) debe repartir la carga entre varias IP o
ejecutarse en un entorno de prueba con los límites documentados.

Antecedentes en el repositorio (del equipo, **no** medidos en esta rama):

- `infraestructura/prueba_carga.py`: N alumnos haciendo registro → fase 1 →
  fase 2 → predicción; informa mediana, p95, p99 y errores por etapa.
- `infraestructura/carga/ejecutar_prueba.py`: escenarios `pico` (K aulas de 45
  alumnos a la vez) y `ciclos` (los mismos alumnos repiten), midiendo CPU por
  cgroup, RAM (`docker stats`) y crecimiento de la base por tabla.
- `infraestructura/carga/resultados/*.json` (septiembre de 2026) y la sección
  de dimensionamiento de `PLAN.md`. Se citarán como referencia y se repetirán
  en el entorno del equipo para tener evidencia propia.

## Qué se va a medir

- **Throughput**: peticiones por segundo y recorridos completos por minuto.
- **Latencia bajo carga**: p50, p95, p99 por etapa del recorrido.
- **Tasa de error**: 5xx, timeouts y 429 (separando los 429 esperados por cupo
  de los que indican saturación).
- **Recursos**: CPU y RAM por contenedor (`revo_auth`, `revo_survey`,
  `revo_ml`, `revo_postgres`, `revo_redis`, `revo_pasarela`), conexiones
  abiertas en PostgreSQL.
- **Punto de quiebre**: número de alumnos simultáneos en el que el p95 o la
  tasa de error superan el criterio.
- **Estabilidad**: comportamiento en una prueba prolongada (`pruebas/README.md`
  menciona una observación de dos horas en staging).

## Herramientas sugeridas

- Scripts existentes del equipo: `infraestructura/prueba_carga.py`
  (`ALUMNOS`, `OLEADAS`) e `infraestructura/carga/ejecutar_prueba.py`
  (`--pico`, `--ciclos`). Se ejecutan tal cual; no se modifican.
- **k6** para escenarios por etapas (`ramping-vus`) con umbrales (`thresholds`)
  y exportación JSON.
- **Locust** como alternativa en Python, con interfaz web útil para la demo
  del laboratorio.
- `docker stats` y consultas a `pg_stat_activity` para recursos y conexiones.

## Plan de casos de prueba

| ID | Tipo | Escenario | Configuración | Qué se registra |
|---|---|---|---|---|
| REN-01 | Humo | 1 aula pequeña | 5 alumnos, recorrido completo, 1 vez | Que el script funciona; 0 errores |
| REN-02 | Carga nominal | 1 aula | 45 alumnos (`ALUMNOS_POR_AULA` de `ejecutar_prueba.py`) llegando en 60 s | p50/p95/p99 por etapa, errores, CPU/RAM |
| REN-03 | Carga alta | Varias aulas a la vez | 3 aulas × 45 (`--pico 3`), cada aula con su IP | Igual que REN-02 |
| REN-04 | Estrés | Subir aulas hasta degradar | `--pico 1,3,6,10` | Punto de quiebre: primer escenario que incumple el criterio |
| REN-05 | Pico de acceso | Login simultáneo | 45 logins en el mismo segundo desde una IP de aula | Latencia de login con bcrypt en paralelo; 429 de la zona `acceso` |
| REN-06 | Ruta cara | Cierre de fase concurrente | 45 `submit_phase` casi simultáneos | p95/máx; ningún corte por `proxy_read_timeout` (35 s); errores 5xx |
| REN-07 | Ciclos | Mismos alumnos repiten | `--ciclos 2,3,4` | Crecimiento de la base por alumno y ciclo; si el historial se vuelve más lento |
| REN-08 | Resistencia | Carga nominal prolongada | REN-02 repetido durante 30–120 min | Fugas de memoria, conexiones abiertas, errores acumulados |
| REN-09 | Limitador | Una sola IP | 200 usuarios virtuales desde la misma IP | Confirmar que aparecen 429 y que no hay 5xx (valida la protección, no la capacidad) |
| REN-10 | Workers | Comparar configuración | REN-03 con `WORKERS=1` y `WORKERS=2` | Diferencia en p95 y CPU |
| REN-11 | Redis caído | Degradación | REN-02 con Redis detenido a mitad | Comportamiento `fail_open` (cupos de alumno dejan pasar; `admin` cierra) sin 5xx masivos |

## Criterio de aceptación (propuesto)

> Propuesto por el equipo de documentación; pendiente de validar con el equipo y el profesor.
> Son umbrales de partida, no resultados.

- **Carga nominal (1 aula, 45 alumnos):** 100 % de recorridos completos,
  p95 global ≤ 1 s, 0 % de errores 5xx.
- **Carga alta (3 aulas, 135 alumnos):** ≥ 99 % de recorridos completos,
  p95 global ≤ 2 s.
- **Estrés:** identificar y documentar el punto de quiebre y el recurso que se
  agota primero (CPU, RAM, conexiones a PostgreSQL).
- **Resistencia:** sin crecimiento sostenido de RAM ni de conexiones durante la
  prueba prolongada.

## Resultados

> Sección vacía a propósito. Se llena solo con pruebas ejecutadas de verdad.
> Copiar una fila por ejecución.

| Fecha | Responsable | Entorno (local / staging, commit) | Caso(s) | Resultado (Aprobado / Fallido / Pendiente) | Evidencia (ruta en `evidencias/` o enlace) |
|---|---|---|---|---|---|
| | | | | | |

Tabla de medidas por escenario (llenar por caso ejecutado):

| Caso | Alumnos | Recorridos completos | p50 (s) | p95 (s) | p99 (s) | % error | CPU pico | RAM pico | Observación |
|---|---|---|---|---|---|---|---|---|---|
| | | | | | | | | | |

### Observaciones

_(Sin ejecuciones registradas todavía.)_
