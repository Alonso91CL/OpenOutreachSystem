# docker

Configuración de Docker para correr OpenOutreach de forma persistente junto al panel web.

Estos archivos viven dentro del repo raíz `OpenOutreachSystem` (no son submódulo). No dependen del código
del fork (`../OpenOutreach`) ni de la interfaz (`../openoutreach-web`).

## Contenido previsto

- `Dockerfile` — imagen propia: `python:3.12-slim` + `openoutreach` + dependencias del panel.
- `docker-compose.yml` — servicio `control` con el volumen de datos y el `.env`.
- `.env.example` — plantilla de variables de entorno.
- `backup.ps1` / `backup.sh` — respaldo de `db.sqlite3` antes de migrar.

## Hechos que condicionan el diseño

- La imagen oficial `ghcr.io/eracle/openoutreach:latest` corre **un job y sale**; no es daemon.
- El estado vive en un único SQLite: `OPENOUTREACH_DB=/app/data/db.sqlite3`.
- La configuración entra 100% por variables `OPENOUTFIND_*` / `OUTSEND_*`.
- **Un solo escritor por base.** El panel debe serializar los jobs.

## Estado

Sin archivos de despliegue todavía. Pendiente la Fase F0 de `../PLAN.md`.
