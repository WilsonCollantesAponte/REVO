# Pruebas no funcionales de REVO

Rama de trabajo: `pruebas` (fork `WilsonCollantesAponte/REVO`).
Curso Integrador II — Ingeniería de Sistemas, UTP Piura.

## Propósito de la rama

La rama `pruebas` es una rama aparte de `main` para **planificar, ejecutar y
evidenciar** las pruebas no funcionales que pidió el profesor, sin tocar el
código que ya está en producción:

| Área | Documento |
|---|---|
| Seguridad | [`seguridad.md`](seguridad.md) |
| Tokens y autenticación | [`tokens-autenticacion.md`](tokens-autenticacion.md) |
| Tiempo de respuesta | [`tiempo-de-respuesta.md`](tiempo-de-respuesta.md) |
| Rendimiento (carga y recursos) | [`rendimiento.md`](rendimiento.md) |
| Bitácora de avances | [`bitacora.md`](bitacora.md) |

Cada commit en esta rama queda fechado y con autor, y sirve como evidencia del
avance (entrega de avance del lunes y correcciones del laboratorio del
miércoles).

## Objetivo de las pruebas

Comprobar con evidencia reproducible que REVO:

1. Resiste los ataques habituales contra una API web (OWASP Top 10 / API Top 10).
2. Emite, valida y rechaza correctamente los tokens de acceso (JWT).
3. Responde en un tiempo aceptable para un alumno en el aula.
4. Soporta la carga de varias aulas simultáneas sin degradarse.

## Qué hay hoy en el código (base de los planes)

Resumen de la arquitectura real, leída del repositorio:

- **Pasarela nginx** (`infraestructura/pasarela/nginx.conf.template`): único
  puerto público. Enruta `/api/auth/`, `/api/legal/` → auth-service;
  `/api/sessions/`, `/api/questions/`, `/api/psychometric/` → survey-service;
  `/api/predict/`, `/api/stats/`, `/api/courses/` → ml-service. Aplica límites
  por IP (`general` 30 r/s, `acceso` 10 r/s para login/registro, 100 conexiones
  por IP), cuerpo máximo 256 KB y cabeceras de seguridad.
- **Tres microservicios FastAPI** (`services/auth-service`,
  `services/survey-service`, `services/ml-service`) construidos sobre la
  librería común `services/comun/revo_comun/` (tokens JWT, rate limit con
  Redis, cabeceras, secreto de pasarela, contexto RLS).
- **PostgreSQL 16 + pgvector** con Row Level Security y un rol por servicio;
  **Redis** para los cupos de peticiones.
- **Despliegue**: `docker-compose.yml` (+ `docker-compose.desarrollo.yml` en
  local, puerto 8080), Coolify en VPS vía GitHub Actions
  (`.github/workflows/ci.yml`: unitarias → E2E frontend → integración →
  construcción Docker → staging → producción, solo desde `main`), y como
  alternativa `render.yaml` (plan free) + `frontend/vercel.json`.
- **Pruebas existentes**: unitarias e integración por servicio
  (`pruebas/README.md`), E2E Playwright en `frontend/pruebas/e2e/`, recorrido
  de sistema en `pruebas/e2e/sistema/flujo_completo.py`, y scripts de carga
  en `infraestructura/prueba_carga.py` e `infraestructura/carga/` (con
  resultados JSON de septiembre de 2026 ya versionados en
  `infraestructura/carga/resultados/`).

Lo que **no** está en el repositorio: la carpeta `database/` con los scripts
SQL que cita el README, y las guías de despliegue y el modelo de amenazas
(el README indica que se mantienen fuera del repositorio público).

## Cómo se organiza

```
docs/pruebas-no-funcionales/
├── README.md                 ← este archivo
├── bitacora.md               ← quién hizo qué y cuándo
├── seguridad.md
├── tokens-autenticacion.md
├── tiempo-de-respuesta.md
├── rendimiento.md
└── evidencias/               ← (se creará) capturas, reportes ZAP, JSON de k6/Locust
```

Cada documento por área tiene la misma estructura:

1. Alcance (servicios y endpoints reales, con ruta de archivo).
2. Qué se va a medir.
3. Herramienta sugerida.
4. Plan de casos de prueba (tabla).
5. Criterio de aceptación **propuesto** (a validar con el equipo y el profesor).
6. Resultados (plantilla vacía: se llena solo con datos medidos de verdad).

> Regla del equipo: en estos documentos **no se escriben resultados que no se
> hayan medido**. Un caso sin ejecutar queda como "Pendiente".

## Flujo de trabajo

1. Trabajar siempre en la rama `pruebas`:
   ```bash
   git clone https://github.com/WilsonCollantesAponte/REVO.git
   cd REVO
   git checkout pruebas
   git pull
   ```
2. Hacer commits pequeños, en español y con estilo convencional, por ejemplo:
   - `docs(pruebas): plan de pruebas de seguridad`
   - `test(carga): escenario k6 de login concurrente`
   - `docs(pruebas): resultados de tiempo de respuesta 13/10`
3. Registrar cada avance en [`bitacora.md`](bitacora.md).
4. Guardar evidencias (reportes, capturas, JSON) en `evidencias/` con fecha en
   el nombre: `AAAAMMDD_area_descripcion.ext`.
5. `git push origin pruebas`.
6. Cuando HebertCG dé acceso de escritura (o acepte PR desde forks), abrir un
   **pull request de `pruebas` hacia `HebertCG/REVO:main`**. El historial de
   commits pasa completo.

> **Ojo con `.gitignore`:** el `.gitignore` de la raíz ignora la carpeta
> `docs/` completa (el equipo guarda ahí guías privadas y el modelo de
> amenazas). Por eso los archivos de esta carpeta se agregaron con
> `git add -f`. Los cambios a archivos ya versionados se suben normal, pero
> **cada archivo nuevo** dentro de `docs/pruebas-no-funcionales/` (por ejemplo
> en `evidencias/`) necesita `git add -f <ruta>`. No se modificó el
> `.gitignore` para no alterar la decisión del equipo; al abrir el PR hacia
> `HebertCG/REVO` hay que acordar con Hebert si esta carpeta se publica o se
> mueve a otra ruta.

Precauciones:

- No commitear `.env`, tokens ni contraseñas reales en evidencias (tapar el
  token en capturas).
- Las pruebas de seguridad y carga se hacen **contra un entorno local o de
  staging propio**, nunca contra producción sin autorización del jefe de grupo.
- No modificar el código de `main` desde esta rama salvo acuerdo del equipo:
  scripts nuevos de prueba van en carpetas nuevas.
