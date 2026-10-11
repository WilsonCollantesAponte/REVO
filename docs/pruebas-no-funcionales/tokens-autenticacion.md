# Pruebas de tokens y autenticación

Estado: **plan** (sin ejecuciones todavía).

## Alcance

### Cómo funciona hoy la autenticación (leído del código)

| Pieza | Archivo | Qué hace |
|---|---|---|
| Emisión y verificación de JWT | `services/comun/revo_comun/seguridad/tokens.py` | Clase `TokenIssuer`. JWT **HS256** con PyJWT. Claims: `sub` (id de usuario como texto de entero), `role` (`student` o `admin`), `iss` (`revo-auth`), `aud` (`revo-api`), `typ` (`access`), `iat`, `exp`, `jti` (uuid4). Lista de algoritmos fija; `exp`, `iat` y `sub` obligatorios; rechaza `typ` distinto de `access`, roles desconocidos y `sub` no entero o ≤ 0 |
| Secreto | `services/comun/revo_comun/seguridad/tokens.py`, `services/comun/revo_comun/ajustes.py`, `.env.example` | `JWT_SECRET` sin valor por defecto; mínimo 32 caracteres; el arranque rechaza valores de ejemplo (`changeme`, `test`, `supersecret`, etc.). Los tres servicios comparten el mismo secreto (`SERVICE_REALBASE64_64_JWT` en `.env.example`) |
| Caducidad | `services/comun/revo_comun/ajustes.py`, `.env.example` | `JWT_EXPIRE_HOURS=24` por defecto |
| Dependencias FastAPI | `services/comun/revo_comun/servicio.py` | `principal` (401 + `WWW-Authenticate: Bearer` si no hay token válido), `admin` (403 si el rol no es `admin`), `principal_opcional`, y `sesion`, que fija el contexto RLS con el `user_id` y el `role` del token |
| Login y registro | `services/auth-service/routers/auth.py` | `POST /auth/register` (201 + token, rol siempre `student`), `POST /auth/login` (mismo mensaje 401 exista o no la cuenta, con verificación de un hash de descarte para igualar tiempos), `POST /auth/login/form` (formulario OAuth2), `GET /auth/verify`, `GET/PUT /auth/me` |
| Contraseñas | `services/auth-service/security.py`, `services/auth-service/config.py` | bcrypt directo, coste 12, truncado explícito a 72 bytes; longitud mínima de contraseña 10 (`PASSWORD_MIN_LENGTH`) |
| Token entre servicios | `services/survey-service/routers/sessions.py` | Al cerrar fase, survey-service **no reenvía** el token del alumno a ml-service: emite uno nuevo con la misma identidad y `expire_seconds=TOKEN_INTERNO_SEGUNDOS` (constante = 180 s; el comentario del código dice "un minuto", hay que confirmar cuál es el valor buscado) |
| Cupos de acceso | `services/auth-service/politicas.py`, `infraestructura/pasarela/nginx.conf.template` | `login`: 8 intentos / 15 min por cuenta (email hasheado); `login_por_ip`: 400 / hora; `register`: 80 / 10 min por IP; `register_diario`: 250 / día por IP. En nginx, zona `acceso` (10 r/s, ráfaga 100) para `/api/auth/login` y `/api/auth/register` |
| Frontend | `frontend/src/services/api.js`, `frontend/src/context/AuthContext.jsx` | El token se guarda en `sessionStorage` (no `localStorage`) y viaja en `Authorization: Bearer`. Ante un 401 el interceptor borra el token |

### Lo que NO existe en el código (dicho tal cual)

- **No hay refresh token**: solo token de acceso de 24 h. Al caducar, el alumno
  vuelve a iniciar sesión.
- **No hay revocación ni lista negra**: el `jti` se genera pero no se consulta
  en ningún sitio. El "cerrar sesión" del frontend (`AuthContext.jsx`) solo
  borra el token del navegador; un token copiado sigue siendo válido hasta su
  `exp`.
- **No hay endpoint de recuperación de contraseña ni de verificación de
  correo** en `services/auth-service/routers/`, aunque los resultados de carga
  listan tablas `password_reset_tokens` y `email_verification_codes` en la
  base (los scripts SQL `database/` no están en el repositorio).
- **No hay cookies de sesión** (`allow_credentials=False` en CORS,
  `services/comun/revo_comun/servicio.py`), así que CSRF clásico no aplica.
- No hay MFA ni OAuth externo (Google, Microsoft, etc.).

Pruebas automatizadas que ya existen y que conviene ejecutar y citar como
evidencia (no se modifican):

- `services/comun/pruebas/unitarias/test_tokens.py` (firma, caducidad, emisor,
  audiencia, `typ`, `alg: none`, `sub` inválido, rol inventado, secreto corto
  o de ejemplo, esquema Bearer, token interno).
- `services/auth-service/pruebas/unitarias/test_security.py` (bcrypt).
- `services/auth-service/pruebas/integracion/test_auth.py` (registro, login,
  enumeración, acceso cruzado, token manipulado, inyección en login).

## Qué se va a medir

- Que cada token alterado, caducado, de otro emisor/audiencia/propósito o sin
  firma sea rechazado con 401 en los **tres** servicios, a través de la pasarela.
- Que el rol del token determine el acceso (alumno vs. admin).
- Que el token interno de survey → ml caduque en el tiempo configurado.
- Comportamiento tras "cerrar sesión" y tras desactivar una cuenta.
- Cupos de login: en qué intento aparece el 429 y qué `Retry-After` devuelve.
- Igualdad de respuesta (código, mensaje y tiempo) entre cuenta inexistente y
  contraseña incorrecta.

## Herramientas sugeridas

- **pytest** (suites existentes: `python -m pytest -q` en `services/comun` y
  `services/auth-service`; integración con `REVO_TEST_DATABASE_URL`, ver
  `pruebas/README.md`).
- **curl** / **HTTPie** contra la pasarela local (`http://localhost:8080/api`).
- **jwt.io** o `python -c "import jwt; ..."` para decodificar y fabricar tokens
  de prueba (solo con secretos de entorno local, nunca el de producción).
- **Burp Suite Community** u **OWASP ZAP** (modo manual) para repetir peticiones
  con tokens modificados.

## Plan de casos de prueba

| ID | Caso | Endpoint | Pasos | Resultado esperado |
|---|---|---|---|---|
| TOK-01 | Login correcto | `POST /api/auth/login` | Credenciales válidas | 200 con `access_token`, `expires_in` = 86400 (24 h) y datos del usuario sin contraseña |
| TOK-02 | Estructura del token | — | Decodificar el token de TOK-01 | Cabecera `alg: HS256`; claims `sub`, `role`, `iss=revo-auth`, `aud=revo-api`, `typ=access`, `iat`, `exp`, `jti` |
| TOK-03 | Token válido en los tres servicios | `GET /api/auth/verify`, `GET /api/sessions/active`, `GET /api/predict/user/{uid}/history` | Mismo token | Respuestas autorizadas (no 401) |
| TOK-04 | Sin cabecera | `GET /api/auth/me` | Sin `Authorization` | 401 con `WWW-Authenticate: Bearer` |
| TOK-05 | Esquema incorrecto | `GET /api/auth/me` | `Authorization: Basic ...` o token sin `Bearer` | 401 |
| TOK-06 | Firma alterada | `GET /api/auth/me` | Cambiar un carácter de la firma | 401 |
| TOK-07 | Payload alterado (escalada) | `GET /api/auth/users` | Cambiar `role` a `admin` sin volver a firmar | 401 |
| TOK-08 | Algoritmo `none` | `GET /api/auth/me` | Token con `alg: none` y sin firma | 401 |
| TOK-09 | Token caducado | `GET /api/auth/me` | Token local con `exp` en el pasado (o `JWT_EXPIRE_HOURS` mínimo en entorno de prueba) | 401 |
| TOK-10 | Otro emisor / audiencia | `GET /api/auth/me` | Token firmado con el secreto local pero `iss` o `aud` distintos | 401 |
| TOK-11 | Otro propósito | `GET /api/auth/me` | `typ` distinto de `access` | 401 |
| TOK-12 | Rol inventado | `GET /api/auth/me` | `role: "superadmin"` firmado con el secreto local | 401 |
| TOK-13 | Alumno en ruta admin | `GET /api/auth/users`, `GET /api/stats/overview`, `GET /api/psychometric/all` | Token `student` válido | 403 |
| TOK-14 | Admin en ruta admin | Mismas rutas | Token `admin` válido | 200 |
| TOK-15 | Registro nunca crea admin | `POST /api/auth/register` | Enviar `role: "admin"` en el cuerpo | Nunca se crea un admin: cuenta con `role: student` o rechazo 422 si el esquema no admite el campo |
| TOK-16 | Igualdad de respuesta en login | `POST /api/auth/login` | 20 intentos con cuenta inexistente y 20 con contraseña incorrecta | Mismo código y mensaje; registrar tiempos de ambos grupos y compararlos |
| TOK-17 | Fuerza bruta por cuenta | `POST /api/auth/login` | Intentos fallidos seguidos contra el mismo email | 429 con `Retry-After` a partir del cupo `login` (8 / 15 min) |
| TOK-18 | Login por formulario | `POST /api/auth/login/form` | Igual que TOK-17 | Documentar el comportamiento (usa otra combinación de cupos, ver SEG-21 en `seguridad.md`) |
| TOK-19 | Token tras cerrar sesión | `GET /api/auth/me` | Copiar el token, cerrar sesión en el frontend, reutilizarlo | Documentar: según el código seguirá siendo válido hasta `exp` (no hay revocación) |
| TOK-20 | Cuenta desactivada | `GET /api/auth/me`, `POST /api/auth/login` | Admin desactiva al alumno con `PATCH /api/auth/users/{id}/toggle` | Login → 403 "Cuenta desactivada"; `/auth/me` con token previo → 401 "Sesion no valida". Documentar qué pasa en survey/ml con ese token previo |
| TOK-21 | Token interno survey → ml | `POST /api/sessions/{id}/submit_phase` | Capturar (en entorno local) el token interno y reutilizarlo después de 180 s | 401 al reutilizarlo pasado su `exp` |
| TOK-22 | Almacenamiento en el navegador | Frontend | Iniciar sesión, cerrar la pestaña y volver a abrir | El token desaparece (`sessionStorage`); nunca en `localStorage` ni en cookies |
| TOK-23 | Contraseña débil | `POST /api/auth/register` | Contraseña de menos de 10 caracteres o igual al correo | 422/400, cuenta no creada |
| TOK-24 | Suites existentes | — | Ejecutar `test_tokens.py`, `test_security.py`, `test_auth.py` | Todas pasan; guardar la salida de pytest como evidencia |

## Criterio de aceptación (propuesto)

> Propuesto por el equipo de documentación; pendiente de validar con el equipo y el profesor.

- 100 % de los tokens inválidos (TOK-04 a TOK-12) rechazados con 401 en los tres servicios.
- 100 % de los casos de rol (TOK-13 a TOK-15) con el resultado esperado.
- Login sin diferencia de mensaje ni de código entre cuenta inexistente y
  contraseña incorrecta; diferencia de tiempo medio a acordar tras medir.
- Cupo de login activo (429 con `Retry-After`).
- Las limitaciones confirmadas (sin revocación, sin refresh) quedan documentadas
  como riesgo aceptado o como mejora propuesta, no ocultas.

## Resultados

> Sección vacía a propósito. Se llena solo con pruebas ejecutadas de verdad.
> Copiar una fila por ejecución.

| Fecha | Responsable | Entorno (local / staging, commit) | Caso(s) | Resultado (Aprobado / Fallido / Pendiente) | Evidencia (ruta en `evidencias/` o enlace) |
|---|---|---|---|---|---|
| | | | | | |

### Observaciones

_(Sin ejecuciones registradas todavía.)_
