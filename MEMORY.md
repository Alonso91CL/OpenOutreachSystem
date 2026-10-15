# MEMORY.md — OpenOutreachSystem

Memoria del proyecto. Estado, decisiones y hechos verificados. **No guardar secretos aquí**
(van en `.env`). Actualizar tras cada decisión importante.

## Identidad

- **Proyecto**: OpenOutreachSystem — prospección B2B con OpenOutreach para **Pivot IT** (`pivotit.cl`).
- **Dueño**: Alonso (`Alonso91CL`).
- **Ruta local**: `C:\Users\Alonso\Documents\Project\OpenOutreachSystem`
- **Remotos**: raíz `github.com/Alonso91CL/OpenOutreachSystem` · panel `github.com/Alonso91CL/openoutreach-web` · fork `github.com/Alonso91CL/OpenOutreach`
- **Creado**: 2026-10-14.

## Herramientas / entorno

- **gh (GitHub CLI)** `2.102.0` instalado global con winget, en PATH de máquina
  (`C:\Program Files\GitHub CLI\gh.exe`).
- Autenticado como `Alonso91CL`: protocolo `https`, token en keyring, scopes
  `gist`, `read:org`, `repo`.
- `git` usa el credential helper de gh para `github.com` y `gist.github.com`.
- Sin scope `workflow`: un push que toque `.github/workflows/` fallará hasta añadirlo
  (`gh auth refresh -s workflow`).

## Estructura actual

```
OpenOutreachSystem\                 repo raíz git (rama develop; main existe)
├── .gitmodules                     submódulo OpenOutreach
├── .gitignore                      ignora .env, data/, state/, openoutreach-web/
├── OpenOutreach\                   submódulo → github.com/Alonso91CL/OpenOutreach (rama main)
├── openoutreach-web\               submódulo → github.com/Alonso91CL/openoutreach-web (main/develop)
├── docker\                         archivos del repo raíz (sin git propio)
├── PLAN.md
├── README.md
├── AGENTS.md
└── MEMORY.md
```

## Decisiones tomadas

| Fecha | Decisión | Motivo |
|-------|----------|--------|
| 2026-10-14 | Tres partes separadas: fork, panel web, docker | Aislar responsabilidades |
| 2026-10-14 | `docker/` NO es submódulo: sus archivos viven en el repo raíz | Capa de despliegue del conjunto |
| 2026-10-14 | `OpenOutreach/` es submódulo del fork del usuario | Seguir upstream y versionar el puntero |
| 2026-10-14 | `openoutreach-web/` declarado como **submódulo** | Repo remoto disponible |
| 2026-10-14 | Proyecto renombrado `llm` → `OpenOutreachSystem` | Nombre definitivo elegido por el usuario |
| 2026-10-14 | Panel en FastAPI + HTMX, consumo por contrato JSON de la CLI | Simplicidad y desacople del esquema interno |

## Hechos verificados de OpenOutreach (análisis 2026-10-14)

- **Tres repos**: OpenOutreach (orquestador, GPLv3), OpenOutFind (búsqueda, GPLv3),
  OpenOutSend (envío, MIT). El orquestador instala y hospeda a los hijos.
- **Pipeline**: descubrir → calificar → puerta de confianza → resolver email → escribir → enviar.
- **Proveedor de datos**: BetterContact Lead Finder. Búsqueda gratis; resolver email cuesta
  **1 crédito** por acierto. Cuenta gratis: 40 créditos sin tarjeta.
- **LLM**: se necesita API key (OpenAI, Anthropic o endpoint compatible).
- **Buzón**: SMTP/IMAP con app password. Google Workspace funciona directo. Requiere SPF/DKIM/DMARC.
- **Estado**: un único SQLite en `~/.openoutreach/data/db.sqlite3`
  (en la imagen Docker: `OPENOUTREACH_DB=/app/data/db.sqlite3`, volumen `/app/data`).
- **Config**: 100% por variables `OPENOUTFIND_*` / `OUTSEND_*`. El wizard solo las recuerda en la DB.
- **Contrato de salida CSV**: `email, first_name, last_name, company, title, website, linkedin_url,
  reason, lead_id, qualified_at`. `reason` = por qué se eligió el lead.
- **Sin web, sin daemon, sin scraping**, cero cuenta de red social. La imagen corre un job y sale.
- **Línea fija `Sent with OpenOutreach`** en cada correo, no removible.
- **Regla dura**: un solo escritor por base SQLite ("one job per database, ever").

## Pivot IT (contexto)

- Agencia TI chilena. Servicios: desarrollo web, apps móviles, Business Intelligence,
  automatización, ciberseguridad, consultoría IT, SEO/SEM.
- Compra típica: gerentes / CIO / CTO de empresas chilenas con procesos manuales o sistemas en silos.
- Web: `pivotit.cl`. LinkedIn: `linkedin.com/company/pivotit-spa`.

## Pendientes / preguntas abiertas

- [x] **Remoto de `openoutreach-web`** configurado y convertido en submódulo.
- [x] **Remoto del repo raíz** configurado y ramas `main` + `develop` subidas.
- [ ] Validar **cobertura de datos en Chile** con `openoutreach find 5` (gratis, no gasta).
- [ ] Decidir alcance del panel: ¿solo búsqueda + CSV, o **envío completo** desde la UI?
- [ ] Elegir siguiente fase: **F0** (Docker funcional) o **F2** (andamiaje del panel).
- [ ] Verificar nombres exactos de variables contra `docs/configuration.md` de cada hijo (F0).

## Riesgos conocidos

- Cobertura de datos en Chile incierta.
- No usar el dominio `pivotit.cl` para cold masivo: preparar SPF/DKIM/DMARC y subdominio dedicado.
- Chile: Ley 21.719 (vigencia dic-2026) y opt-out obligatorio.
- UI expuesta sin auth = cualquiera gasta créditos o envía correo.
