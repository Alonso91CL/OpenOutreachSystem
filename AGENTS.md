# AGENTS.md — OpenOutreachSystem

Instrucciones para agentes (OpenCode, Claude Code, Codex, cualquier otro) que trabajen en este
proyecto. Leer junto con `PLAN.md` (qué se construye) y `MEMORY.md` (estado y decisiones).

## Qué es este proyecto

Poner OpenOutreach a funcionar para la prospección B2B de **Pivot IT** (`pivotit.cl`): búsqueda de
leads, calificación, exportación y envío, más un panel web de control y su configuración Docker.

- Directorio de trabajo: `C:\Users\Alonso\Documents\Project\OpenOutreachSystem`
- Idioma de trabajo y de respuesta: **español**.

## Mapa del repositorio

| Ruta | Naturaleza | Regla |
|------|-----------|-------|
| `OpenOutreach/` | **submódulo** → `github.com/Alonso91CL/OpenOutreach` | No modificar sin motivo. Si se toca, se hace en el fork y se sube el puntero del submódulo en la raíz |
| `openoutreach-web/` | **submódulo** → `github.com/Alonso91CL/openoutreach-web` | Trabajo del panel; gitflow propio |
| `docker/` | archivos del repo raíz | Aquí van Dockerfile, compose y `.env.example` |
| `PLAN.md`, `README.md`, `AGENTS.md`, `MEMORY.md` | archivos del repo raíz | Mantener actualizados |

## Reglas de git

- Modelo **gitflow** en la raíz y en `openoutreach-web`: ramas `main` (producción) y `develop`
  (integración). **Nunca** commit directo a `main` ni `develop`.
- Trabajo nuevo: `git flow feature start <nombre>` … `git flow feature finish <nombre>`.
- Commits: Conventional Commits, una línea. Tipos `feat fix docs refactor test chore perf build ci`.
- **Preguntar antes de `git push`** y antes de cualquier comando destructivo.
- Reportar rama + resumen del diff en stage antes de cada commit.
- El submódulo `OpenOutreach` sigue su rama `main`; su upstream publica release en cada push a `main`.

## Reglas de trabajo

- **Verificar antes de afirmar.** Correr el comando; reportar qué se saltó o no se pudo verificar,
  con el comando exacto y la línea de error decisiva.
- **Sin refactors ni cambios no pedidos** en archivos que no toca la tarea.
- **Paquete Python**: si hiciera falta Node en el panel, usar **pnpm** (nunca npm/yarn).
- Actualizar `MEMORY.md` cuando se tome una decisión importante; actualizar `PLAN.md` si cambia el plan.

## Reglas técnicas del dominio (OpenOutreach)

- **La UI consume la CLI por su contrato JSON** (`openoutreach find N --json`, `find 0 --json`,
  `status --json`). **Nunca** leer tablas internas de SQLite: el esquema pertenece a los paquetes hijos.
- **Un solo escritor por base SQLite.** Serializar todos los jobs; jamás dos procesos escribiendo a la vez.
- **Gastar créditos y enviar correo son irreversibles.** Confirmación explícita obligatoria en la UI;
  `find` sin `emails` es gratis.
- **Secretos solo en `.env`** (permisos 600, fuera de git). Nunca en la UI, la base ni los logs.
- Cada correo que envía la herramienta termina con la línea fija `Sent with OpenOutreach`, no removible.

## Comandos de referencia

```bash
# Submódulos
git submodule update --init --recursive      # traer el fork tras un clone
git submodule update --remote OpenOutreach   # actualizar el fork a su main

# OpenOutreach (una vez instalado / en Docker)
openoutreach status --json
openoutreach find 5                # gratis, sin gastar
openoutreach find 5 emails        # ≤5 créditos
openoutreach send

# Docker (cuando exista docker/docker-compose.yml)
docker compose -f docker/docker-compose.yml up -d
```

## Alcance / no-goals

- No reescribir ni "forkear" `openoutfind` / `openoutsend` dentro de este proyecto.
- No añadir pipeline nuevo: el panel es una capa de operación alrededor de la CLI.
- No exponer el panel a internet sin autenticación y TLS.
