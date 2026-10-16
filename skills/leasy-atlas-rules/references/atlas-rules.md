# Atlas — Reglas

Ámbito exclusivo: proyecto Leasy Atlas, repositorio `leasyauto/atlas`. No aplicar estas reglas a appops, appopsbackend, dryft.pe, leasy_ops_backoffice, nexode ni a ningún otro proyecto.
Stack: Django + Pydantic + Django-Ninja + Inertia + Svelte + Storybook + i18n con `translate` / `translate_for_request`.

---

# Regla Suprema: Aprobación Humana Obligatoria para GitHub

> [!CAUTION]
> **REGLA ABSOLUTA #0:**
> **NADA LLEGA A GITHUB SIN LA APROBACIÓN EXPLÍCITA DE LUCAS.**
>
> - **Prohibido ejecutar `git push` del código de Atlas sin autorización** (a cualquier rama, sea `feature`, `fix` o `main`).
> - **Prohibido crear Pull Requests** (`gh pr create`).
> - **Prohibido editar Pull Requests** (`gh pr edit`).
> - **Prohibido publicar comentarios o modificar issues en GitHub**.
>
> Las mutaciones en GitHub requieren la autorización de Lucas. Actualización expresa del 14 de octubre de 2026: las capturas de UI de Atlas se suben automáticamente a `lucast1574/pr-assets` siguiendo [capturas y assets del PR](pr-screenshots.md). Esta excepción no autoriza push de código, crear/editar PRs, comentarios ni issues. El código y el cuerpo preparado del PR permanecen locales hasta que Lucas autorice su publicación.
> cuando se pida mirar un comentario ir a por el comentario directamente no hay necesidad de correr test o mas cosas ir a ver el comentario en el pr defrente para empezar a corregir
> si t pido hacer un ticket no empiezes a correr test y demas haz el ticket el codigo y demas analisis para el ticket y despues corres los test no antes

## Regla maestra: honestidad estructural

Cada línea de código debe declarar honestamente lo que hace.

- Si algo puede fallar, que falle visible.
- Si algo tiene un contrato, que sea único y explícito.
- Si algo es regla de negocio, que viva en un solo lugar y se enforce donde el atacante no llega.
- Nunca ocultar un problema para que la app parezca sana.

Ante cualquier duda: _¿estoy ocultando algo? ¿Estoy repitiendo una decisión que ya vive en otro lado? ¿Estoy dejando dos versiones de la verdad?_

---

## 1 · Fail-fast, sin defensiva

- Vistas y APIs son la única frontera hostil. Validan con Pydantic al entrar.
- Servicios y dominio confían en el payload validado. Prohibido:
  - Fallbacks inline: `is_active = get_status(v) or True`.
  - Coerción manual: `_parse_bool`, `try_parse_int`.
  - Casts defensivos en Svelte sobre props ya tipadas (`Boolean(prop)`, `Number(prop)`).
  - Revalidar lo que Pydantic ya validó.
- Excepciones estrechas: capturar solo la excepción cuyo comportamiento se va a manejar. `except Exception` y `except OSError` amplios están prohibidos como handlers de dominio.
- Si algo debe fallar, falla. Preferible una excepción visible a un fallback que devuelva `None`, `False`, `0`, o un placeholder.
- Frontend Svelte: si un prop llega mal tipado, dejar fallar y arreglar el backend. No parchear en el componente.

## 2 · Una sola fuente de verdad

Cada dato de dominio vive en un solo lugar y las demás capas lo derivan:

- Constantes de negocio: solo en Python. El backend expone el valor ya resuelto. Frontend consume, no recalcula.
- Límites de tamaño/tiempo (`MAX_*`): declarados una vez y todas las capas derivan del mismo símbolo.
- Traducciones: registradas atómicamente en `keys.ts`, `atlas/i18n/keys.py` (`I18NKeys.all()`), `locale/default/LC_MESSAGES/django.po` y todos los `.po` de países activos. Si falta una capa, la clave no existe.
- Estado de un flujo, decisión de negocio, permiso, cálculo monetario: enforcado en un servicio único. Nunca duplicar la regla en el cliente TS.
- URLs del frontend: helper del middleware Inertia, nunca hardcoded.

## 3 · Contratos honestos en fronteras

- Fechas cross-boundary: `datetime` con timezone-aware, nunca `str`.
- IDs de terceros: se acompañan del emisor (`external_id` + `external_source`), nunca solos.
- Base64 y campos grandes: límites de request declarados en el mapeo por-endpoint, no en el default global.
- HTTP status: cada código diferencia un caso. 202 ≠ 201 ≠ 204; el cliente assertar el código exacto, no un rango.
- Respuestas con `response.ok`: chequear el flag antes de asumir el tipo.
- Schemas de recepción no son schemas de emisión: separar `*Read` de `*Write`, `*Request` de `*Response`.
- Nombres de rutas técnicas en inglés. Palabras en español pertenecen a la UI.

## 4 · Enforce en el servidor, nunca en el cliente

- Las reglas de negocio se enforcan en el servicio, no en el frontend.
- Un cliente HTTP puede omitir un parámetro, mandar campos extra, o llamar directo al service. La validación debe seguir sirviendo.
- Los flags booleanos de obligación se llaman `requires_*` (obligatorio) o `supports_*` (acepta el path opcional). No `two_sided=True` porque es ambiguo.
- Los permisos multi-tenant se verifican en la vista y también dentro del servicio si puede llamarse por otras rutas.
- Ocultar un botón en la UI no reemplaza la validación server-side.

## 5 · Servicios reciben IDs, no instancias ORM

- Los servicios reciben `record_id: int` o `UID`. Nunca instancias de Django.
- Si el servicio necesita el objeto, hace `Model.objects.get(pk=id)` fresco adentro.
- Django cachea las instancias: pasar una instancia entre capas causa data desactualizada y efectos secundarios.
- Aplica en: services, tasks async, signals, dispatchers, helpers, callbacks de dominio.

## 6 · Single-shot > estado intermedio

Cuando algo se pueda resolver en (A) un solo request atómico o (B) flujo multi-paso con estado server-side (staging en S3, tokens opacos, discard endpoints, TTLs, prune commands), la opción por defecto es (A).

(B) se justifica solo si el ticket lo pide explícitamente por escrito:

- Reanudar tras cerrar la pestaña / cambiar de dispositivo.
- Procesar asíncronamente cargas grandes en background.

(A) elimina staging huérfano, race conditions entre pasos, discard endpoints, prune commands, tests que blindarían el orden peligroso.

## 7 · Tests protegen invariantes, no orden

- El test correcto es _"el frontal no se pierde si la persistencia falla"_ o _"el PDF final tiene exactamente 2 páginas"_.
- El test incorrecto es _"después del merge se borra el staging"_ — blinda el orden peligroso de un flujo que no debería existir.
- Regla mental: si el test asegura un orden de efectos secundarios, protege un flujo peligroso. Si asegura una propiedad del resultado, protege una invariante real.
- Cada test debe morir contra `main` si el feature del PR no está aplicado. Si el mismo test pasa en la branch base sin mi diff, es un falso positivo.
- Prohibido ajustar el `assert` (relajar de `==` a `in`, quitar campos, cambiar el sujeto) solo para que pase.
- `pytest.raises(Exception)` acepta `AttributeError`, imports rotos, `KeyError`. Especificar la excepción exacta.
- Fixtures que reproducen manualmente lo que hace el workflow real: ejecutar el workflow, no una copia.
- Tests que asserten `status_code != 400 or "expected_string" not in body` pasan con 404, 409 o 500. Assertar `status_code == 201` exacto.
- Tests de migraciones son ruido. Las migraciones se validan corriendo `migrate` sobre DB limpia + los tests de la feature.

## 8 · Concurrencia sobre estado compartido

- `select_for_update()` no bloquea filas que aún no existen. Para reservar un registro nuevo, usar `INSERT ... ON CONFLICT` o un lock aplicativo.
- Dos requests que leen `remaining_amount` en paralelo pueden validar cada uno "hay suficiente" y sobregirar el requerimiento. Bloquear el requerimiento, no solo el pago.
- Callbacks de integraciones externas que retroceden estado: proteger la transición hacia atrás con un lock + verificación del estado actual antes de commitear.
- Estado mutado en variables de módulo Python (`Image.MAX_IMAGE_PIXELS`, dicts globales, defaults mutables) causa races entre requests concurrentes. Nunca mutar globals por-request.

## 9 · i18n y copy multipaís

- Toda cadena visible al usuario pasa por `translate` / `translate_for_request`.
- Cero fallbacks en las llamadas: `translate(msgs, KEY)` sin tercer argumento. El fallback oculta claves faltantes.
- Claves nuevas registradas en las 4 capas (`keys.ts`, `I18NKeys`, `default.po`, todos los `.po` de países activos) o no van.
- Copy de negocio no vive en constantes de Python ni en helpers TS. Vive en catálogos i18n.
- Rutas técnicas (URLs, códigos de país, códigos de moneda) en inglés y sin acentos.
- Ningun texto puede ser hardcoded tod i18n.

## 10 · Money, tiempo y unidades

- Dinero: siempre con moneda explícita. Nunca sumar montos brutos y "aplicar conversión después". Convertir en la frontera.
- Tasas de cambio: origen + destino + fecha. Un `Decimal` suelto no dice si multiplica o divide.
- Fechas para presentación: formatear en la timezone del usuario, no del host.
- `new Date("YYYY-MM-DD")` en TS interpreta como UTC y luego lee en local: la fecha cambia según la zona del navegador.
- Reglas de moneda cross-country: cero fallbacks silenciosos (`return {}` cuando falta la config). Fallar visible.
- Un pago `validated` sin conversión no debe presentarse como total confiable. Si falta conversión, el total va como "no disponible", no como suma parcial.

## 11 · Migraciones son sagradas

- Estructura, no data. `RunPython` que rellena JSON, borra rows por scope o backfillea campos calculables va en un management command aparte.
- Una migración por feature. Colapsar antes del PR con `makemigrations` fresh.
- Nunca aceptar sin cuestionar una migración auto-generada. `makemigrations --dry-run` primero; comparar; regenerar si difiere.
- YAGNI aplica a schemas: no agregar columnas "por si acaso".
- Merge migrations permitidas pero preferido: rename lineal. Si `main` trae `0005_A` y mi branch `0005_B`, renombrar la mía a `0006_B` y ajustar `dependencies`. Cero `0006_merge_...py` cosmético.
- Cero tests dedicados a la migración misma.
- Antes de push: `makemigrations --check --dry-run` con `DJANGO_SETTINGS_MODULE=atlas.core.settings.test` y `migrate --plan` lineal.
- Migraciones y force-push solo con OK explícito.

## 12 · Layering y ubicación de la lógica

- `views/` = frontera. Valida entrada, serializa salida con `.model_dump()`. Cero `if` de negocio, cero queries directas al ORM.
- `services/` = dominio. Reciben IDs, confían en el payload validado, enforcan la regla.
- `schemas/` = contratos Pydantic frozen. Uno de entrada, otro de salida cuando la simetría no es exacta.
- `constants/` = valores compartidos entre capas, expuestos al frontend serializados.
- `apps/**/*.py`: nada de `django.forms` en código nuevo. Todo Pydantic.
- Frontend `src/components/**`: NO tocar salvo pedido explícito. La UI trabaja con props del backend; los cambios de UI se proponen, no se aplican.
- Si toco un `.svelte` de `src/components/`, en el mismo commit va el `.stories.svelte`.

---

## Uploads y artefactos server-side

- Single POST con todas las partes; el componente mantiene los `File` en memoria.
- Cero `_*_staging/`, cero `staging_id`, cero discard endpoints, cero prune commands, cero TTL.
- Validación por firma real de bytes usando el helper existente (`atlas.core.utils.upload_validation.validate_uploaded_file` + `UploadRules`). Nunca confiar en `content_type` declarado.
- Catálogo de INPUT ≠ catálogo de ARTEFACTO. Si el usuario sube JPG/PNG y el backend produce un PDF combinado, el catálogo de input NO acepta PDF solo para que "pase" el artefacto downstream.
- Flags: `requires_*_upload` / `supports_*_upload`, enforcados en el service.
- Validar propiedades del artefacto antes de persistir: PDFs con páginas exactas, no cifrados, no corruptos. Imágenes: EXIF corregido, dimensiones capadas ANTES de descomprimir para evitar decompression bomb.
- Límites: una sola fuente para `MAX_SIDE_SIZE_MB`, `MAX_COMBINED_SIZE_MB`, `HTTP_ROUTE_LIMIT_MB`, con margen documentado.
- `except` estrecho: solo la excepción específica.
- Nunca `Image.MAX_IMAGE_PIXELS` global; chequear `width * height` local antes de `image.load()`.
- Cero persistencia de docs de identidad en `localStorage` / `IndexedDB`.
- Tests HTTP-level con Django `Client` cubren: auth/scope negativo, body size en frontera, archivo corrupto, multipart malformado, éxito crea exactamente un `LeadDocument`.

## Integraciones externas y callbacks

- Auth asimétrico si el canal es bidireccional: bearer distinto por dirección, documentado.
- `PUBLIC_BASE_URL` obligatorio antes de enviar. Si falta, fallar antes del POST; nunca enviar rutas relativas.
- Callback: proteger transiciones hacia atrás. Si Atlas marca `completed` y llega un ack tardío de `dispatched`, ignorar sin revertir.
- Idempotencia con clave explícita en el schema; sin ella, no procesar retries.
- Estado de fila que Ops nunca vuelve a tocar: crear un job periódico o mecanismo de expiración explícito. Comentar "se limpia en X" sin implementar X es una promesa vacía.
- Sentry PII: `response.text` de un 4xx puede traer DNI/teléfono. Sanitizar antes de reportar.

## Kanban / Inertia / Svelte pages

- Props del backend salen de `.model_dump()`; los tipos TS los reflejan sin castear.
- Cero botones habilitados en UI si el permiso backend no los expone. Menú completo, opciones deshabilitadas visibles.
- Botón `<button>` NUNCA dentro de `<a>` / `<Link>`. HTML inválido.
- Layouts con `overflow-hidden` no anidan más `overflow-hidden` en descendientes que necesitan scroll.
- Componentes de `src/components/` no se modifican salvo pedido explícito.
- Story creada/actualizada al tocar `src/components/*.svelte`.
- Errores de fetch: siempre chequear `response.ok`. Un 500 no debe caer al `catch` como si fuera error de red.
- Loops de N peticiones secuenciales que escalan con estado del usuario: rediseñar a un solo request agregado.

## Workflows y máquinas de estado

- Una etapa/tab/router de workflow tiene UNA fuente de verdad. Cero listados paralelos.
- Tab inválido → 404 explícito, no fallback silencioso al default.
- Orden de guards importa: descarte por cliente/aval antes de observaciones bloqueantes cambia la salida. Documentar orden en la definición del workflow.
- `on_enter_*` que devuelve `skipped` cuando falta configuración crítica: si el enter es única vez, ese skip es permanente. Fallar visible.

## Datos personales, aprobaciones, PII

- Payload de aprobación contiene exactamente lo que se aprueba, no todo el snapshot.
- Deduplicación de solicitudes por (lead + tipo + hash del payload), no solo por lead.
- IDs de catálogo no se comparan por nombre visible. Los distritos con nombre repetido rompen la comparación.
- Un fingerprint no debe fabricarse en tests reemplazando parcialmente un campo. Snapshot completo o test que muestre el bug.
- Blacklist: `token_set_ratio` da falsos positivos con nombres cortos comunes. Umbral por país y por longitud.
- Auditoría de identidad: guardar payload original + payload usado para decidir, no uno en función del otro.

---

## Entrega: commits, PRs, comentarios

### Comentarios en el código

- Máximo 2 líneas. Preferible cero.
- Un fix de 1 línea NUNCA lleva bloque de 6 líneas explicando el porqué.
- Tests no llevan docstring salvo que el nombre del test no baste.
- Cero comentarios obvios (`# create user`), cero headings de sección (`# --- Helpers ---`), cero "invariantes" arriba de un `if` obvio.
- Cero docstrings de 3-8 líneas en helpers internos. Si el nombre no basta, mejorar el nombre.
- Comment SOLO cuando: (a) el código hace algo contraintuitivo, (b) `# noqa` puntual con motivo, (c) TODO/FIXME con owner+ticket.
- Docstrings module-level también cuentan. Un docstring de 20 líneas explicando el flujo es ruido en un repo con PRs.

### Commit messages

- Conventional Commits en el subject: `fix(scope): descripción corta`.
- Sin body. Si el reviewer quiere detalle va al PR body.
- Cero "Reglas aplicadas: ..." en el commit ni en el PR body.
- Nunca meter el nombre del reviewer en el subject.

### PR body

Si cambia la UI visible, incluir la captura verificada con el enlace de GitHub de `lucast1574/pr-assets`, siguiendo [capturas y assets del PR](pr-screenshots.md). Prepararla automáticamente; la publicación del cuerpo respeta la autorización del PR.

Estructura:

1. Contexto breve.
2. Qué hace y cuándo.
3. Cómo funciona (pseudocódigo o snippet si aporta).
4. Cambios agrupados.
5. Tests que corrí.
6. QA manual si aplica.

Nada de "reglas aplicadas", nada de justificaciones extra. Los reviewers leen el código; el body es contexto.

### Respuestas a review

- Punto por punto, en el orden del reviewer.
- Cita textual del reviewer en blockquote arriba de cada respuesta cuando aporta claridad.
- Cambios concretos con `archivo:línea` o snippet de código.
- Nombre exacto del test que protege el cambio.

### Branches y push

- `atlas` usa PRs. Crear `feature/<tema>` desde la branch activa. `gh pr create --base <activa> --head <feature>`.
- Migraciones y force-push: solo con OK explícito.
- Antes de push: `ruff check src/`, `pytest` de los archivos tocados y sus vecinos, `makemigrations --check`, `bun run check`, `bun run lint:ox`.

---

## Cómo aplicar esto en la práctica

Antes de escribir código:

1. ¿Qué contrato voy a romper o crear?
2. ¿Dónde vive la regla de negocio y qué me obliga a duplicarla?
3. ¿Qué falla puede quedar oculta si esto sale mal?
4. ¿Tengo un test que protege la propiedad del resultado, no el orden de pasos?
5. ¿Este cambio necesita servicio, o cabe en la vista?
6. ¿Este cambio pasa por single-shot o estoy inventando estado intermedio?

Antes de commit:

1. Correr los checks locales.
2. Leer el diff completo con ojos de reviewer.
3. Borrar comentarios que agregué "para explicar". El código habla.
4. No mandar nada a github sin la aprobacion de lucas

Antes de responder a un review:

1. Reproducir el caso que el reviewer plantea. Si no puedo, preguntarle sin tirar teorías.
2. Fixear el fondo, no la superficie.
3. Escribir la respuesta punto por punto, con archivo:línea.
4. Si un cambio abre nueva superficie, agregar el test que la protege.

# Atlas — Desarrollo local

Guía para el PC de Lucas. Todo corre via Docker Compose.

Ubicación del repo: `D:\vscode\leasyauto\atlas`

---

## Primer arranque (una sola vez)

Docker Desktop tiene que estar abierto y con el ícono verde en la barra de tareas.

Desde una PowerShell abierta en tu sesión (no SSH):

```powershell
Set-Location D:\vscode\leasyauto\atlas

# 1. Pull imágenes base (~500MB)
docker pull postgres:17
docker pull redis:7

# 2. Build imágenes del proyecto (5-15 min primera vez, uv sync + bun install)
docker compose build

# 3. Levantar todo
docker compose up -d

# 4. Esperar Postgres healthy (30-60s)
docker compose logs postgres --tail 20

# 5. Migraciones
docker compose exec web python src/manage.py migrate

# 6. Superuser local (introducir contraseña de forma interactiva)
$env:DJANGO_SUPERUSER_EMAIL = "admin@atlas.test"
docker compose exec web python src/manage.py createsuperuser --email admin@atlas.test
```

O corre el script `setup-atlas-docker.ps1` que hace todo (pasos 1-6) automático.

---

## Uso diario

```powershell
# Prender
docker compose up -d

# Ver estado
docker compose ps

# Ver logs
docker compose logs -f web           # backend Django
docker compose logs -f frontend      # vite + svelte
docker compose logs -f worker        # dramatiq tasks
docker compose logs -f postgres

# Apagar (mantiene DB)
docker compose stop

# Apagar y borrar todo (⚠️ pierde datos)
docker compose down -v

# Reiniciar un servicio
docker compose restart web

# Rebuild después de cambiar Dockerfile o pyproject.toml
docker compose build web && docker compose up -d web
```

---

## URLs locales

| Servicio       | URL                         |
| -------------- | --------------------------- |
| Backend Django | http://localhost:8000       |
| Django Admin   | http://localhost:8000/admin |
| Frontend Vite  | http://localhost:5173       |
| Postgres       | localhost:5432              |
| Redis          | localhost:6379              |

---

## Superuser default (fake, solo para test local)

| Campo          | Valor                       |
| -------------- | --------------------------- |
| Email          | `admin@atlas.test`          |
| Password       | Elegida interactivamente al crear el usuario |
| Admin Django   | http://localhost:8000/admin |
| Frontend Atlas | http://localhost:5173       |

Este correo es fake (dominio `.test` reservado, nunca resolverá DNS). Solo sirve para login local.

Si quieres crear otro superuser manualmente (te va a pedir email + password interactivamente):

```powershell
docker compose exec web python src/manage.py createsuperuser
```

Recomendado también crear un usuario operativo (no-admin) para probar el flujo de UI real desde el frontend.

---

## Conexión a la base de datos

El proyecto es multi-país. Con `COUNTRY_CODE=pe` en el `.env`, Django usa la DB `atlas_pe`.

Postgres 17 corre en el container `atlas-postgres-1` con el puerto mapeado a tu host.

### Conexión desde herramientas externas (DBeaver, TablePlus, psql local)

Desde tu host:

| Campo    | Valor                                                        |
| -------- | ------------------------------------------------------------ |
| Host     | `localhost`                                                  |
| Puerto   | `5432`                                                       |
| Database | `atlas_pe` (o `atlas_mx`, `atlas_ec`, `postgres` para admin) |
| User     | `leasy_user`                                                 |
| Password | ver `.env` → `POSTGRES_PASSWORD`                             |
| SSL      | disabled                                                     |

### Cadenas de conexión

```
# App activa (PE)
postgresql://leasy_user:<POSTGRES_PASSWORD>@localhost:5432/atlas_pe

# Admin (crear/dropear DBs)
postgresql://leasy_user:<POSTGRES_PASSWORD>@localhost:5432/postgres
```

### DBs disponibles

El entrypoint del container crea automáticamente:

| DB         | Owner        | Para qué                   |
| ---------- | ------------ | -------------------------- |
| `postgres` | `leasy_user` | DB admin, no usar para app |
| `atlas_pe` | `atlas_pe`   | Perú (default en `.env`)   |
| `atlas_mx` | `atlas_mx`   | México                     |
| `atlas_ec` | `atlas_ec`   | Ecuador                    |

Reporter user read-only (para dashboards SQL):

| Campo    | Valor                                          |
| -------- | ---------------------------------------------- |
| User     | `reporter_user`                                |
| Password | ver `.env` → `POSTGRES_PASSWORD`               |
| Permisos | SELECT-only en `public` de cada `atlas_<país>` |

### Cambiar de país

```powershell
# Editar .env → COUNTRY_CODE=mx (o ec)
# Reiniciar el backend
docker compose restart web worker scheduler
```

Cada país tiene su propia DB, sus propias migraciones, su propio catálogo.

---

## Comandos frecuentes de Django

```powershell
# Migraciones
docker compose exec web python src/manage.py migrate
docker compose exec web python src/manage.py makemigrations --check --dry-run
docker compose exec web python src/manage.py migrate --plan

# Shell interactiva
docker compose exec web python src/manage.py shell

# Seed data (si aplica)
docker compose exec web python src/manage.py populate_fake_data

# Tests
docker compose exec web pytest src/atlas/apps/leads -x --tb=short
docker compose exec web pytest src/atlas/apps/users -x
COUNTRY_CODE=ec docker compose exec -e COUNTRY_CODE=ec web pytest ...

# Compilar traducciones (después de tocar .po)
docker compose exec web python src/manage.py compilemessages
```

---

## Comandos frecuentes de Postgres

```powershell
# psql interactivo dentro del container
docker compose exec postgres psql -U leasy_user -d atlas_pe

# Query one-shot
docker compose exec postgres psql -U leasy_user -d atlas_pe -c "SELECT COUNT(*) FROM leads_lead;"

# Dump de una DB
docker compose exec postgres pg_dump -U leasy_user atlas_pe > backup_pe.sql

# Restore
Get-Content backup_pe.sql | docker compose exec -T postgres psql -U leasy_user -d atlas_pe

# Listar todas las DBs y roles
docker compose exec postgres psql -U leasy_user -d postgres -c "\l"
docker compose exec postgres psql -U leasy_user -d postgres -c "\du"

# Reset total de una DB (⚠️ irreversible)
docker compose exec postgres psql -U leasy_user -d postgres -c "DROP DATABASE atlas_pe;"
docker compose exec postgres psql -U leasy_user -d postgres -c "CREATE DATABASE atlas_pe OWNER atlas_pe;"
docker compose exec web python src/manage.py migrate
```

---

## Estructura del proyecto

```
D:\vscode\leasyauto\atlas\
├── src/
│   ├── manage.py                   # Django entry
│   └── atlas/
│       ├── apps/                   # Django apps por dominio (leads, users, entities, ...)
│       │   └── <app>/
│       │       ├── models.py
│       │       ├── services/       # Lógica de negocio (reciben IDs, no instancias)
│       │       ├── views/          # Frontera HTTP (Pydantic in, model_dump out)
│       │       ├── schemas/        # Pydantic schemas
│       │       ├── constants.py
│       │       ├── factories/      # factory_boy para tests
│       │       ├── migrations/
│       │       └── tests/          # pytest
│       ├── core/
│       │   ├── settings/
│       │   │   ├── base.py
│       │   │   ├── develop.py      # DJANGO_SETTINGS_MODULE default
│       │   │   ├── test.py         # para pytest
│       │   │   └── production.py
│       │   ├── schemas/            # money, common
│       │   └── utils/
│       ├── frontend/               # Svelte + Vite + Storybook
│       │   ├── src/
│       │   │   ├── pages/          # Páginas Inertia
│       │   │   │   ├── Leads/
│       │   │   │   ├── User/
│       │   │   │   └── Role/
│       │   │   ├── components/     # NO tocar salvo pedido explícito
│       │   │   └── i18n/keys.ts    # Enum de i18n
│       │   └── package.json
│       ├── i18n/keys.py            # Enum Python de i18n
│       └── locale/                 # Catálogos .po
│           ├── default/
│           ├── es_PE/
│           ├── es_MX/
│           ├── es_EC/
│           └── es_SV/
├── docker-compose.yml
├── Dockerfile                      # target dev usa uv + hot reload
├── pyproject.toml                  # Python 3.13, uv managed
├── uv.lock
└── .env                            # Config local
```

---

## Frontend (Svelte + Vite)

El frontend corre dentro del container `frontend` con `bun run dev` en el puerto 5173. Hot reload automático al editar `.svelte` o `.ts`.

```powershell
# Storybook (si necesitas)
docker compose exec frontend bun run storybook   # puerto 6006

# Type check
docker compose exec frontend bun run check

# Lint
docker compose exec frontend bun run lint:ox

# Tests
docker compose exec frontend bun run test
```

---

## Debug de problemas comunes

### Backend no responde en :8000

```powershell
docker compose logs web --tail 50
# Buscar: ImproperlyConfigured, MigrationSchemaMissing, connection refused
```

### Frontend no compila

```powershell
docker compose logs frontend --tail 50
docker compose exec frontend bun install
```

### Postgres se queda en "waiting for connections"

Esperar 30-60s la primera vez. Verificar:

```powershell
docker compose exec postgres pg_isready -U leasy_user -d atlas_pe
```

### Reset total del stack

```powershell
docker compose down -v      # elimina volumes = pierde DB
docker compose build --no-cache
docker compose up -d
docker compose exec web python src/manage.py migrate
docker compose exec web python src/manage.py createsuperuser
```

---

## Notas de arquitectura

- **Multi-país**: cada país tiene su propia DB. Cambiar `COUNTRY_CODE` en `.env` y reiniciar.
- **Servicios reciben UIDs, no instancias ORM**: `service.method(record_id=x)` siempre.
- **Vistas usan Pydantic**: `Schema.model_validate()` para in, `.model_dump(mode="json")` para out. Cero `django.forms` en código nuevo.
- **i18n atómico**: nueva key registrada simultáneamente en `atlas/i18n/keys.py`, `frontend/src/i18n/keys.ts`, `locale/default/LC_MESSAGES/django.po` y todos los `.po` de países activos.
- **Frontend `src/components/`**: no tocar salvo pedido explícito.
- **Migraciones**: una por feature. `makemigrations --check --dry-run` local antes de commit. Cero `RunPython` para llenar data.

Reglas completas en `C:\Users\lucas\Downloads\ATLAS-RULES.md`.

---

## Git

```powershell
# Branch actual
git branch --show-current

# Trabajar en una branch nueva (para PR)
git checkout -b feature/KAN-XXX-descripcion
# Editar código...
git add <archivos>
git commit -m "fix(scope): descripción corta"
git push -u origin feature/KAN-XXX-descripcion
gh pr create --base main --head feature/KAN-XXX-descripcion --title "..." --body-file ..
```

Convención de commits:

- `fix(<scope>): descripción corta`
- Sin body. Nunca meter nombre del reviewer.
- Nunca `apply KAN-XXX review (Nombre)`. Correcto: `apply KAN-XXX review`.
- Siempre el titulo en español
