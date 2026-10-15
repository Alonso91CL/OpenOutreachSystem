# OpenOutreachSystem

Repositorio principal del proyecto de prospección con OpenOutreach, orientado a Pivot IT.
Aquí viven la configuración de despliegue, el plan y los submódulos.

## Estructura

| Ruta | Qué es | Cómo se versiona |
|------|--------|------------------|
| `OpenOutreach/` | Fork del orquestador (upstream `eracle/OpenOutreach`) | **submódulo** → `https://github.com/Alonso91CL/OpenOutreach.git` |
| `openoutreach-web/` | Panel web de control (FastAPI + HTMX) | **submódulo** → `https://github.com/Alonso91CL/openoutreach-web.git` |
| `docker/` | Configuración Docker (imagen, compose, `.env`) | archivos de **este** repo raíz |
| `PLAN.md` | Plan de Docker + panel web | archivos de **este** repo raíz |
| `AGENTS.md` | Instrucciones para agentes | archivos de **este** repo raíz |
| `MEMORY.md` | Estado y decisiones del proyecto | archivos de **este** repo raíz |

## Repositorios remotos

- **Raíz**: `https://github.com/Alonso91CL/OpenOutreachSystem.git` — ramas `main` + `develop`
- **Panel**: `https://github.com/Alonso91CL/openoutreach-web.git` — ramas `main` + `develop`
- **Orquestador (fork)**: `https://github.com/Alonso91CL/OpenOutreach.git` — rama `main`

## Notas de diseño

- **Submódulos**: se actualizan con `git submodule update --remote <ruta>`, sin mezclar el trabajo
  del panel con el fork.
- **`docker/`** no es submódulo: sus archivos son de este repo y forman la capa de despliegue.
- El panel **consume la CLI por su contrato JSON** (`find --json`, `status --json`), nunca lee tablas
  internas de SQLite.
- Los dos repos nuevos usan **gitflow** (`main` + `develop`); el fork sigue su `main` trunk-based.

## Estado

- [x] Fork declarado como submódulo en `OpenOutreach/`.
- [x] Panel `openoutreach-web/` declarado como submódulo.
- [x] Remotos configurados y ramas `main` + `develop` subidas.
- [ ] Validar cobertura de datos en Chile con `openoutreach find 5` (gratis).
- [ ] F0 Docker funcional · F2 andamiaje del panel.

Ver `PLAN.md` para fases, riesgos y decisiones pendientes.
