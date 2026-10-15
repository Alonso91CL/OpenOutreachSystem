# Plan: OpenOutreach en Docker + panel web de control

> Objetivo: correr OpenOutreach de forma persistente en Docker y manejarlo desde una
> interfaz web sencilla (buscar leads, ver resultados, enviar, ver estado), sin tocar
> los paquetes internos (`openoutfind`, `openoutsend`).

Estado: **plan**. No incluye el código todavía. Define alcance, arquitectura, pasos y riesgos.

---

## 1. Contexto y restricciones que condicionan el diseño

Hechos del repositorio que el plan debe respetar:

1. **OpenOutreach no tiene superficie web.** Es una CLI (`find`, `send`, `status`, `run`),
   sin `URLconf`, sin Django Admin, sin puertos. La app web es **algo que nosotros
   agregamos alrededor**, nunca dentro del paquete.
2. **La imagen oficial ya existe**: `ghcr.io/eracle/openoutreach:latest`. Corre **un job acotado y
   sale**; no es un daemon.
3. **El estado es un único SQLite**: `~/.openoutreach/data/db.sqlite3` (en la imagen,
   `OPENOUTREACH_DB=/app/data/db.sqlite3`, montado en `/app/data`). Ahí viven leads, empresas,
   deals, el log de correo y las respuestas del onboarding.
4. **Regla dura: "one job per database, ever".** SQLite con WAL permite lecturas concurrentes,
   pero **un solo escritor**. La app web debe **serializar** cualquier job que escriba.
5. **Los hijos (finder, sender) leen su configuración desde variables de entorno**
   (`OPENOUTFIND_*`, `OUTSEND_*`) en cada corrida. El wizard solo las escribe en la DB y las
   exporta. Un instalación headless se configura 100% por entorno.
6. **Dos acciones gastan o son irreversibles**: resolver email (créditos) y enviar correo
   (no se puede deshacer). La UI debe exigir confirmación explícita en ambas.
7. **Frontera estable para la UI**: `openoutreach find N --json`, `openoutreach find 0 --json` y
   `openoutreach status --json` son contratos públicos. **La UI debe consumir esos comandos**,
   no leer tablas internas, para no romperse cuando los hijos cambien de esquema.

---

## 2. Alcance

### Dentro

- Docker Compose con la imagen de OpenOutreach + un servicio de panel web.
- Panel web sencillo: estado, leads, exportar CSV, lanzar `find`/`send`, ver log en vivo.
- Configuración por archivo `.env` (secretos fuera de la UI y de la DB).
- Override de la secuencia de correo para escribir **en español**.
- Un solo trabajador de jobs (cola interna) para respetar el escritor único de SQLite.
- Autenticación mínima y bind a localhost / red privada.

### Fuera (no-goals)

- No reescribir ni "forkear" `openoutfind` / `openoutsend`.
- No multiusuario, no roles, no multi-tenant.
- No editar leads dentro de la DB (la única escritura pasa por la CLI).
- No eliminar la línea "Sent with OpenOutreach" (no es configurable; ver riesgo en §8).
- No reemplazar el CLI: el panel es una capa de operación.

---

## 3. Arquitectura

Un solo contenedor de aplicación (imagen propia, basada en Python) que instala
`openoutreach` + el panel web, y **lanza la CLI como subproceso**. Un volumen comparte el estado.

```
                ┌─────────────────────────────────────────────┐
                │  contenedor: openoutreach-control            │
                │                                             │
  navegador ────┤  FastAPI + Uvicorn  (panel, puerto 8080)    │
   (HTMX)       │      │                                       │
                │      │  cola de jobs (1 worker, asyncio)      │
                │      ▼                                       │
                │  subprocess: `openoutreach find|send|status` │
                │      │                                       │
                └──────┼───────────────────────────────────────┘
                       │  (env var: OPENOUTREACH_DB, OPENOUTFIND_*, OUTSEND_*)
                       ▼
              volumen ./data/db.sqlite3   (único escritor = la CLI)
              volumen ./sequences/        (secuencia en español)
```

Alternativa descartada: servicio separado "runner" + socket de Docker. Añade privilegios y
complejidad. El subproceso dentro del mismo contenedor basta y respeta el escritor único.

### Por qué FastAPI y no Django Admin

Los hijos ya traen Django Admin (útil para el CRM de OpenOutFind), pero **OpenOutreach no lo expone**
y montarlo implicaría un segundo proyecto Django. Un panel ligero (FastAPI + Jinja2 + HTMX) es
más simple, no depende del esquema interno y habla con la CLI por su contrato JSON.

---

## 4. Componentes

### 4.1 Dockerfile (imagen propia)

Base `python:3.12-slim`. Pasos:

1. `pip install openoutreach` (trae ambos hijos, Django incluido).
2. `pip install fastapi uvicorn jinja2 python-multipart` (dependencias del panel).
3. Copiar el código del panel (`/app/web`).
4. `USER` no-root; `WORKDIR /app`.
5. `ENV OPENOUTREACH_DB=/app/data/db.sqlite3`, `OUTSEND_HOME=/app/state`.
6. `CMD ["uvicorn", "web.main:app", "--host", "0.0.0.0", "--port", "8080"]`.

### 4.2 docker-compose.yml

Servicios mínimos:

```yaml
services:
  control:
    build: .
    env_file: .env
    environment:
      OPENOUTREACH_DB: /app/data/db.sqlite3
      OUTSEND_HOME: /app/state
    volumes:
      - ./data:/app/data                 # leads + log de correo
      - ./state:/app/state               # secuencias + estado del sender
      - ./content:/content               # product.md, target.md
    ports:
      - "127.0.0.1:8080:8080"            # solo localhost; detrás de proxy/TLS si se expone
    restart: unless-stopped
```

Opcional: servicio `scheduler` (mismo imagen) que corre `openoutreach find ...`/`send` por
cron, pero **apagado por defecto** para no competir por el escritor. Si se activa, debe pasar por la
misma cola o por un lock de archivo. Mejor: que el propio panel tenga un scheduler interno opcional
(§6, fase 5).

### 4.3 Volúmenes y contenido

- `./data` → DB. Es lo único crítico. **Respaldar antes de migrar** (`find` aplica migraciones al entrar).
- `./state` → donde el sender guarda su secuencia cold. Override en español en §5.
- `./content` → `product.md` y `target.md`. Se pueden editar desde la UI (fase 5) o a mano.

### 4.4 Configuración (.env)

Un archivo `.env` (permisos 600, fuera de git) con:

```
OPENOUTFIND_PRODUCT_DOCS_FILE=/content/product.md
OPENOUTFIND_CAMPAIGN_TARGET_FILE=/content/target.md
OPENOUTFIND_LLM_API_KEY=...
OPENOUTFIND_LLM_MODEL=...
OPENOUTFIND_BETTERCONTACT_API_KEY=...
OPENOUTFIND_OPERATOR_EMAIL=...
OPENOUTFIND_OPERATOR_COUNTRY=CL
OUTSEND_AI_MODEL=...
OUTSEND_LLM_API_KEY=...
OUTSEND_MAILBOX_ADDRESS=...
OUTSEND_MAILBOX_PASSWORD=...          # app password
OUTSEND_OPERATOR_NAME=...
OUTSEND_OPERATOR_EMAIL=...
OUTSEND_SIGNATURE=...
PANEL_USER=...
PANEL_PASSWORD_HASH=...
```

> Nota: los nombres exactos de variables deben confirmarse contra
> `docs/configuration.md` de cada hijo antes de cerrar el `.env`. Es un paso de la Fase 0.

---

## 5. Secuencia de correo en español

El opener y los pasos se escriben desde archivos de prompt Jinja. El cold sequence viene en
inglés. Para escribir en español:

1. Copiar la carpeta cold a `./state/sequences/cold/`.
2. Reescribir los `.prompt.j2` en español, manteniendo el nombre (`E0-opener.prompt.j2`, etc.)
   porque **el nombre es el calendario**.
3. Montar `./state` en el contenedor.

> Verificar en Fase 0 la ruta real de override bajo el orquestador: OpenOutreach fija
> `OUTSEND_HOME` al state dir, así que la ruta puede ser `<state>/sequences/cold/` y no
> `~/.openoutsend/sequences/cold/`. Confirmar antes de montar.

---

## 6. Panel web (MVP y fases)

### Pantallas MVP

1. **Estado**: `openoutreach status --json` → configurado sí/no, leads por estado, correos
   enviados, créditos estimados, último run.
2. **Leads**: tabla desde `openoutreach find 0 --json` (todas las filas, sin gasto). Columnas clave:
   empresa, cargo, nombre, email, `reason`. Filtro por texto y por "con email / sin email".
   Botón **Exportar CSV** (`find 0` → stdout).
3. **Ejecutar**: botones
   - `Buscar N (gratis)` — no gasta.
   - `Buscar N con email (gasta ≤N créditos)` — confirmación obligatoria.
   - `Enviar` — confirmación obligatoria ("pone correo a desconocidos, irreversible").
   - Input numérico con tope duro; botón **Detener trabajo**.
4. **Log en vivo**: streaming (SSE) de stdout/stderr del subproceso.
5. **Historial de jobs**: hora, verbo, parámetros, código de salida, resultado.

### Backend

- `POST /jobs` {verb, args} → encola. Un worker asyncio procesa **uno a la vez** y guarda
  `state = {idle|running|failed|done}`.
- `GET /events` → SSE con `{line, stream}` del job actual.
- `GET /leads` → parseo de `find 0 --json` (con caché corta).
- `GET /status` → `status --json`.
- `GET /export.csv` → descarga del CSV.
- `POST /stop` → termina el subproceso actual.

### Controles anti-error (crítico)

- Lock global: si hay un job corriendo, cualquier otro se rechaza.
- Toda acción con gasto o envío: modal de confirmación con el costo/irreversibilidad escrito.
- No exponer los secretos: `.env` se muestra enmascarado y solo lectura.
- Timeout por job y límite de créditos por corrida.

---

## 7. Fases de implementación

| Fase | Entregable | Criterio de done |
|------|-----------|------------------|
| **F0** | Línea base Docker | Imagen oficial corre un job; volumen persiste; `.env` completo y verificado; probar `find 5` real |
| **F1** | Secuencia en español | Un opener generado sale en español; ruta de override confirmada |
| **F2** | Backend solo-lectura | `GET /status`, `GET /leads`, `GET /export.csv` responden desde la CLI |
| **F3** | Trigger de jobs + SSE | Lanzar `find 5` desde la UI, ver log en vivo, cola bloquea doble ejecución |
| **F4** | Envío + confirmaciones | `send` desde la UI con modal; botón detener funciona |
| **F5** | Auth, edición de contenido, scheduler opcional | Login activo; editar `product.md`/`target.md`; cron interno opcional |
| **F6** | Empaque y respaldos | Compose final documentado; script de backup de la DB; README de operación |

---

## 8. Riesgos y mitigaciones

| Riesgo | Impacto | Mitigación |
|--------|---------|-----------|
| SQLite: dos escritores a la vez | DB corrupta / "database is locked" | Cola de 1 worker; scheduler opcional desactivado por defecto |
| Acoplar la UI a tablas internas | Rompe al actualizar hijos | Consumir solo `--json` de la CLI, no el esquema |
| Gasto accidental de créditos | Costo real | Confirmación + tope duro + `find` gratis por defecto |
| Envío accidental | Irreversible, reputación | Confirmación + preview del opener antes de enviar |
| Secretos filtrados | Cuenta comprometida | `.env` 600, UI enmascara, sin logs de secretos |
| Panel expuesto a internet | Cualquiera gasta/envía | Bind a localhost; proxy con TLS + auth si se expone |
| Línea "Sent with OpenOutreach" | Marca / fricción con prospecto | No es removible. Evaluar: usar solo `find` + enviar con herramienta propia |
| Reputación del dominio `pivotit.cl` | Correos a spam | SPF/DKIM/DMARC, calentamiento, tope diario bajo, subdominio dedicado |
| Migraciones en cada arranque | Pierde DB | Backup automático de `db.sqlite3` antes de cada `find` |
| Cobertura de datos en Chile | Pocos leads | Validar con `find 5` gratis antes de invertir en el panel |

---

## 9. Legal (Chile)

- **Tú eres el responsable del correo** (la herramienta da el mecanismo, no cobertura legal).
- Chile: Ley 21.719 (nueva de protección de datos, vigencia dic-2026) y normativa de
  comunicaciones. Los correos comerciales deben poder detenerse (opt-out).
- La herramienta ya aporta línea de baja, cabecera `List-Unsubscribe` y lista de supresión.
- Antes de enviar en volumen: revisar SPF/DKIM/DMARC del dominio y no usar el dominio principal
  para cold outreach.

---

## 10. Decisión abierta para el usuario

¿Panel **read-only + find gratis** primero (sin envío desde el panel), o panel con **envío
completo** desde el inicio? La primera opción elimina el riesgo legal y de marca mientras se
valida la cobertura de datos en Chile.
