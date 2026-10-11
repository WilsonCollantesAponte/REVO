# Pruebas de tiempo de respuesta

Estado: **plan** (sin ejecuciones todavía).

Este documento mide **cuánto tarda cada petición** con poca carga (un alumno o
pocos a la vez). El comportamiento con muchos alumnos simultáneos está en
[`rendimiento.md`](rendimiento.md).

## Alcance

Recorrido real de un alumno, endpoint por endpoint, siempre **a través de la
pasarela** (`infraestructura/pasarela/nginx.conf.template`), que es lo que ve el
navegador:

| Paso del alumno | Endpoint público | Servicio y archivo | Observaciones del código relevantes para el tiempo |
|---|---|---|---|
| Registro | `POST /api/auth/register` | auth · `services/auth-service/routers/auth.py` | Calcula un hash bcrypt de coste 12 (`services/auth-service/security.py`, el comentario estima ~250 ms en hardware modesto) |
| Inicio de sesión | `POST /api/auth/login` | auth · `routers/auth.py` | bcrypt también cuando la cuenta no existe (hash de descarte) |
| Perfil / sesión viva | `GET /api/auth/me`, `GET /api/auth/verify` | auth · `routers/auth.py` | Lectura simple con RLS |
| Documentos legales | `GET /api/legal/documents` | auth · `routers/legal.py` | Público |
| Abrir cuestionario | `POST /api/sessions/` | survey · `services/survey-service/routers/sessions.py` | |
| Preguntas de la fase | `GET /api/sessions/{id}/questions` | survey · `routers/sessions.py` | Motor adaptativo en `services/survey-service/motor_adaptativo/` |
| Guardar respuestas | `POST /api/sessions/{id}/answers` | survey · `routers/sessions.py` | Un solo commit por lote |
| Cerrar fase | `POST /api/sessions/{id}/submit_phase` | survey → ml · `routers/sessions.py` | Llama por red interna a `ml-service /predict/` con `ML_TIMEOUT_SECONDS=90` (`services/survey-service/config.py`); nginx corta la lectura a los 35 s (`proxy_read_timeout`). Es la petición más cara |
| Resultado | `GET /api/predict/{prediction_id}` | ml · `services/ml-service/routers/predict.py` | |
| Cursos de la rama | `GET /api/courses/specialization/{spec_id}` | ml · `routers/courses.py` | |
| Historial | `GET /api/predict/user/{uid}/history` | ml · `routers/predict.py` | Últimas 10 predicciones |
| Realimentación | `POST /api/predict/{prediction_id}/feedback` | ml · `routers/predict.py` | |
| Panel admin | `GET /api/stats/overview`, `GET /api/stats/export-csv` | ml · `routers/stats.py` | Solo admin |
| Salud | `GET /health` | pasarela (respuesta fija de nginx) | Línea base sin backend |

Antecedentes en el repositorio (del equipo, **no** medidos en esta rama):
`infraestructura/prueba_carga.py` ya calcula mediana, p95 y p99 por etapa, y
`PLAN.md` (sección de dimensionamiento del VPS) recoge mediciones anteriores.
Se usarán como referencia de comparación, no como resultados de este plan.

Fuera de alcance: el tiempo de carga del frontend en el navegador
(Lighthouse) se puede añadir después si el profesor lo pide.

## Qué se va a medir

Por endpoint, con N repeticiones (propuesto N = 30) y un solo usuario:

- Tiempo total de respuesta: mínimo, mediana (p50), p95, máximo.
- Tiempo hasta el primer byte (TTFB) donde la herramienta lo permita.
- Código HTTP (para descartar que un tiempo bajo sea un error rápido).
- Arranque en frío: primera petición tras levantar los contenedores (o tras
  dormir, si se prueba en Render `plan: free`, ver `render.yaml`).

Condiciones a registrar siempre: máquina (CPU/RAM), entorno (local con
`docker-compose.yml` + `docker-compose.desarrollo.yml`, o staging), commit
probado, `WORKERS`, y si Redis estaba activo.

## Herramientas sugeridas

- **k6** (`http_req_duration`, `http_req_waiting` = TTFB; escenario de 1 VU con
  N iteraciones) — recomendado por exportar JSON fácil de versionar.
- **curl** con `-w "%{time_starttransfer} %{time_total}\n"` para mediciones
  puntuales y reproducibles.
- **Apache Bench (ab)** o **hey** para repetir un mismo endpoint GET.
- Alternativa en Python: reutilizar la lógica de `infraestructura/prueba_carga.py`
  con `ALUMNOS=1` (sin modificar el archivo; copiar a una carpeta nueva si hace
  falta cambiarlo).
- Pestaña **Network** de las herramientas de desarrollo del navegador para
  evidencia visual del recorrido completo.

## Plan de casos de prueba

| ID | Caso | Endpoint | Pasos | Resultado esperado (a medir) |
|---|---|---|---|---|
| TR-01 | Línea base de la pasarela | `GET /health` | 30 peticiones, 1 usuario | Registrar p50/p95; sirve para restar la latencia de red |
| TR-02 | Registro | `POST /api/auth/register` | 30 altas con correos distintos | Registrar p50/p95; código 201 |
| TR-03 | Login correcto | `POST /api/auth/login` | 30 logins | Registrar p50/p95; código 200 |
| TR-04 | Login fallido | `POST /api/auth/login` | 30 intentos con contraseña incorrecta (sin superar el cupo de 8 por cuenta: usar varias cuentas) | Registrar p50/p95; comparar con TR-03 |
| TR-05 | Lecturas autenticadas | `GET /api/auth/me`, `GET /api/sessions/active` | 30 peticiones cada uno | Registrar p50/p95 |
| TR-06 | Preguntas de la fase | `GET /api/sessions/{id}/questions` | 30 peticiones en sesiones activas | Registrar p50/p95 |
| TR-07 | Guardado de respuestas | `POST /api/sessions/{id}/answers` | Lote de respuestas de una fase, 30 veces | Registrar p50/p95 |
| TR-08 | Cierre de fase con predicción | `POST /api/sessions/{id}/submit_phase` | 30 cuestionarios completos (respetando el cupo `submit_phase`: 20 / 5 min por alumno) | Registrar p50/p95/máx; confirmar que ninguno se acerca al corte de 35 s de nginx |
| TR-09 | Resultado y cursos | `GET /api/predict/{id}`, `GET /api/courses/specialization/{spec_id}` | 30 peticiones cada uno | Registrar p50/p95 |
| TR-10 | Historial | `GET /api/predict/user/{uid}/history` | Con 1 y con 10 predicciones previas | Registrar p50/p95 y comparar |
| TR-11 | Exportación admin | `GET /api/stats/export-csv` | 5 peticiones con token admin | Registrar tiempo y tamaño de la respuesta |
| TR-12 | Arranque en frío | Primer `POST /api/auth/login` tras `docker compose up` | 3 repeticiones | Registrar tiempo de la primera petición vs. las siguientes |
| TR-13 | Recorrido completo | Registro → test → resultado → historial | 10 recorridos con 1 alumno | Registrar tiempo total del recorrido y por paso |

## Criterio de aceptación (propuesto)

> Propuesto por el equipo de documentación; pendiente de validar con el equipo y el profesor.
> Son umbrales de partida, no resultados.

- Lecturas (`GET`) con un solo usuario: **p95 ≤ 500 ms** en entorno local.
- Login y registro (incluyen bcrypt): **p95 ≤ 1 s**.
- Cierre de fase con predicción (`submit_phase`): **p95 ≤ 2 s** y **ninguna**
  petición por encima del `proxy_read_timeout` de 35 s.
- 0 % de errores 5xx durante las mediciones.

## Resultados

> Sección vacía a propósito. Se llena solo con pruebas ejecutadas de verdad.
> Copiar una fila por ejecución.

| Fecha | Responsable | Entorno (local / staging, commit) | Caso(s) | Resultado (Aprobado / Fallido / Pendiente) | Evidencia (ruta en `evidencias/` o enlace) |
|---|---|---|---|---|---|
| | | | | | |

Tabla de medidas (llenar por caso ejecutado):

| Caso | N | p50 (ms) | p95 (ms) | máx (ms) | % error | Observación |
|---|---|---|---|---|---|---|
| | | | | | | |

### Observaciones

_(Sin ejecuciones registradas todavía.)_
