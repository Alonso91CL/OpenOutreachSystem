# llm

Repositorio principal del proyecto de prospección con OpenOutreach, orientado a Pivot IT.
Aquí viven la configuración de despliegue, el plan y los submódulos.

## Estructura

| Ruta | Qué es | Cómo se versiona |
|------|--------|------------------|
| `OpenOutreach/` | Fork del orquestador (upstream `eracle/OpenOutreach`) | **submódulo** → `https://github.com/Alonso91CL/OpenOutreach.git` |
| `openoutreach-web/` | Panel web de control (FastAPI + HTMX) | repo git propio. Pendiente de URL remota; se agregará como **submódulo** |
| `docker/` | Configuración Docker (imagen, compose, `.env`) | archivos de **este** repo raíz |
| `PLAN.md` | Plan de Docker + panel web | archivos de **este** repo raíz |

## Notas de diseño

- **`OpenOutreach/`** es un submódulo que apunta al fork. Se actualiza con
  `git submodule update --remote OpenOutreach`, sin mezclarse con el trabajo del panel.
  Su rama `main` sigue el upstream, que publica una release en cada push a `main`.
- **`openoutreach-web/`** es una aplicación nueva. Consume la CLI por su contrato JSON
  (`find --json`, `status --json`), nunca lee tablas internas. Vive como repo separado y
  se integrará como submódulo cuando tenga remoto.
- **`docker/`** no es submódulo: sus archivos forman parte de este repo y son la capa de
  despliegue del conjunto.
- **La raíz `llm` no anida git repos sin declarar.** El panel queda en `.gitignore`
  mientras no sea submódulo, para que git no lo trate como repo embebido.

## Estado

- [x] Fork clonado y declarado como submódulo en `OpenOutreach/`.
- [x] `openoutreach-web/` renombrado e inicializado con git (gitflow: `main` + `develop`).
- [x] `docker/` integrado como archivos del repo raíz.
- [ ] Validar cobertura de datos en Chile con `openoutreach find 5` (gratis).
- [ ] F0 Docker funcional · F2 andamiaje del panel.

Ver `PLAN.md` para fases, riesgos y decisiones pendientes.
