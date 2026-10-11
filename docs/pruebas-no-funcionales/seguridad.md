# Pruebas de seguridad

Estado: **plan** (sin ejecuciones todavía).

## Alcance

Superficie pública real según el código:

| Componente | Archivo | Qué expone |
|---|---|---|
| Pasarela nginx | `infraestructura/pasarela/nginx.conf.template` | Único puerto público; rutas `/api/*`, frontend estático, `/health` |
| auth-service | `services/auth-service/routers/auth.py` | `POST /auth/register`, `POST /auth/login`, `POST /auth/login/form`, `GET/PUT /auth/me`, `GET /auth/verify`, `GET/PUT /auth/me/consents`, `GET /auth/users` (admin), `PATCH /auth/users/{user_id}/toggle` (admin) |
| auth-service | `services/auth-service/routers/legal.py` | `GET /legal/documents`, `GET /legal/documents/{doc_type}` |
| survey-service | `services/survey-service/routers/sessions.py` | `POST /sessions/`, `GET /sessions/active`, `GET /sessions/{session_id}/questions`, `POST /sessions/{session_id}/answers`, `POST /sessions/{session_id}/submit_phase`, `GET /sessions/` |
| survey-service | `services/survey-service/routers/questions.py`, `psychometric.py` | `GET /questions/`, `/questions/categories/list`, `/questions/{id}`, `GET /psychometric/specialization/{spec_id}`, `GET /psychometric/all` (admin) |
| ml-service | `services/ml-service/routers/predict.py` | `POST /predict/`, `GET /predict/{prediction_id}`, `GET /predict/user/{uid}/history`, `GET /predict/model/importances` (admin), `GET /predict/model/resumen` (admin), `POST /predict/{prediction_id}/feedback` |
| ml-service | `services/ml-service/routers/stats.py`, `courses.py` | `POST /stats/train`, `GET /stats/overview`, `/stats/training-history`, `/stats/export-csv` (todos admin), `GET /courses/specialization/{spec_id}` |

Controles de seguridad que el código declara (y que hay que **verificar**, no
dar por hechos):

- Cabeceras (`services/comun/revo_comun/seguridad/cabeceras.py`):
  `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy`,
  CSP restrictiva, `Permissions-Policy`, `Cache-Control: no-store`, HSTS solo
  en producción; nginx repite cabeceras y `server_tokens off`.
- Límite de cuerpo 256 KB (app: `LimiteTamanoMiddleware`; nginx:
  `client_max_body_size 256k`) y timeouts anti-Slowloris en nginx.
- Secreto de pasarela `X-Revo-Gateway` (`seguridad/pasarela.py`): los servicios
  responden 403 si falta, salvo `/health`, `/healthz`, `/ready`.
- Documentación OpenAPI cerrada en producción (`revo_comun/servicio.py`) y
  bloqueada en nginx (`location ~ ^/api/.*(docs|redoc|openapi\.json)` → 404).
- Métodos permitidos en nginx: GET, POST, PUT, PATCH, DELETE, OPTIONS (resto 405).
- CORS con lista exacta de orígenes, `allow_credentials=False` (`.env.example`, `servicio.py`).
- Rate limit por alumno/IP/credencial (`revo_comun/limites/`, `politicas.py` de cada servicio) y zonas `limit_req` en nginx.
- RLS en PostgreSQL y un rol por servicio (los scripts SQL `database/` no están en el repo; solo se ve el uso desde `revo_comun/basedatos/`).
- Contraseñas con bcrypt coste 12 (`services/auth-service/security.py`).
- Imágenes Docker de servicios ejecutan como usuario `revo` (verificado en CI, `.github/workflows/ci.yml`).

Fuera de alcance: el agente de cursos (`services/ml-service/agente_cursos/`,
servicio `agente-cursos` del compose) salvo su aislamiento de red, y la
infraestructura del VPS/Coolify.

## Qué se va a medir

- Presencia y valor de cabeceras de seguridad en respuestas 2xx, 4xx y 429/502.
- Rechazo de acceso directo a microservicios sin el secreto de pasarela.
- Control de acceso: alumno vs. admin vs. anónimo; acceso a recursos de otro alumno (IDOR).
- Validación de entradas: inyección SQL, payloads enormes, tipos inválidos.
- Exposición de información: mensajes de error, versión del servidor, docs OpenAPI.
- Enumeración de usuarios en login/registro.
- Alertas del escaneo automático (OWASP ZAP) clasificadas por riesgo.

## Herramientas sugeridas

- **OWASP ZAP** (baseline y API scan contra la pasarela local `http://localhost:8080`).
- **curl** / **HTTPie** para casos manuales reproducibles (guardar comando y salida).
- **pytest + httpx** para automatizar casos de control de acceso (nueva carpeta, sin tocar las suites existentes).
- **sqlmap** solo en entorno local, sobre parámetros concretos.
- `pip-audit` y `npm audit` para dependencias (`services/*/requirements.txt`, `frontend/package-lock.json`).

## Plan de casos de prueba

| ID | Caso | Endpoint / componente | Pasos | Resultado esperado |
|---|---|---|---|---|
| SEG-01 | Cabeceras de seguridad en respuesta normal | `GET /api/auth/verify`, `/` | `curl -i` con token válido | Están las cabeceras listadas en Alcance; sin `Server` con versión |
| SEG-02 | Cabeceras en errores de nginx | Cualquier ruta forzando 429 o 404 | Superar `limit_req` o pedir `/api/inexistente` | Cabeceras presentes también en 429/404 (`add_header ... always`) |
| SEG-03 | Acceso directo a microservicio sin pasarela | auth/survey/ml en red Docker o URL Render | Petición sin `X-Revo-Gateway` | 403 "Acceso denegado." (con `REQUIRE_GATEWAY=true`) |
| SEG-04 | Rutas libres de pasarela | `/health` de cada servicio | Petición directa | 200 `{"status":"ok"}` sin datos de versión |
| SEG-05 | Docs OpenAPI cerradas | `/api/auth/docs`, `/api/predict/openapi.json` | GET por la pasarela | 404 |
| SEG-06 | Método no permitido | `/api/auth/me` | `TRACE` / `TRACK` | 405 |
| SEG-07 | Cuerpo excesivo | `POST /api/sessions/{id}/answers` | Cuerpo > 256 KB | 413 |
| SEG-08 | Alumno accede a ruta admin | `GET /api/auth/users`, `POST /api/stats/train`, `GET /api/stats/export-csv` | Token de rol `student` | 403 |
| SEG-09 | Anónimo accede a ruta protegida | `GET /api/auth/me`, `POST /api/predict/` | Sin `Authorization` | 401 |
| SEG-10 | IDOR en historial | `GET /api/predict/user/{uid}/history` | Alumno A pide `uid` de B | 403 |
| SEG-11 | IDOR en predicción y sesión | `GET /api/predict/{prediction_id}`, `GET /api/sessions/{session_id}/questions` | Alumno A usa id de B | No devuelve datos de B (403/404) |
| SEG-12 | Escalada en perfil | `PUT /api/auth/me` | Enviar `role: "admin"`, `is_active` | Campos ignorados; rol sin cambios |
| SEG-13 | Enumeración en login | `POST /api/auth/login` | Email existente con clave mala vs. email inexistente | Mismo código (401) y mismo mensaje |
| SEG-14 | Enumeración en registro | `POST /api/auth/register` | Email ya registrado | Documentar respuesta (el código devuelve "El email ya esta registrado": evaluar riesgo) |
| SEG-15 | Inyección SQL | `POST /api/auth/login` (email), parámetros de ruta numéricos | Payloads `' OR 1=1--` etc. | 401/422, sin error 500 ni datos |
| SEG-16 | Desactivar la propia cuenta admin | `PATCH /api/auth/users/{id}/toggle` | Admin con su propio id | 400 |
| SEG-17 | CORS con origen no permitido | Cualquier `/api/*` | `Origin: https://evil.example` | Sin `Access-Control-Allow-Origin` para ese origen |
| SEG-18 | Falsificación de IP | Rutas con cupo por IP | Enviar `X-Forwarded-For` falso | El cupo no se reinicia (`ip_cliente.py`, `TRUSTED_PROXY_COUNT`) |
| SEG-19 | Escaneo automático | Pasarela completa | ZAP baseline + API scan | Sin alertas de riesgo Alto |
| SEG-21 | Cupos de `POST /auth/login/form` | `POST /api/auth/login/form` | Repetir intentos fallidos contra una misma cuenta y comparar con `/api/auth/login` | Documentar comportamiento. En el código, esta ruta solo aplica el cupo `login` sin `por_credencial=True` (`services/auth-service/routers/auth.py`) y en nginx cae en la zona `general` de `/api/auth/`, no en `acceso`: verificar si la protección anti fuerza bruta es equivalente |
| SEG-20 | Dependencias vulnerables | requirements y package-lock | `pip-audit`, `npm audit` | Sin vulnerabilidades críticas sin justificar |

## Criterio de aceptación (propuesto)

> Propuesto por el equipo de documentación; pendiente de validar con el equipo y el profesor.

- 0 alertas de riesgo **Alto** en OWASP ZAP; las de riesgo Medio documentadas con justificación o plan.
- 100 % de los casos de control de acceso (SEG-08 a SEG-12) con el resultado esperado.
- Ninguna respuesta 500 ante entradas maliciosas (SEG-15).
- Cabeceras de seguridad presentes en el 100 % de las respuestas muestreadas.

## Resultados

> Sección vacía a propósito. Se llena solo con pruebas ejecutadas de verdad.
> Copiar una fila por ejecución.

| Fecha | Responsable | Entorno (local / staging, commit) | Caso(s) | Resultado (Aprobado / Fallido / Pendiente) | Evidencia (ruta en `evidencias/` o enlace) |
|---|---|---|---|---|---|
| | | | | | |

### Observaciones

_(Sin ejecuciones registradas todavía.)_
