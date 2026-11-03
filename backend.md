# R40 — Plan de trabajo del backend (Parte 1 · API en Python)

Nov 2, 2026 · @Ignacio

Este documento baja a tareas concretas la **Parte 1 — API en Python** del enunciado (`R40 — Nueva página web e-commerce.md`). Divide el trabajo en **etapas** y cada etapa en **commits de no más de 500 líneas**, con sus **tests unitarios** y sus **peticiones de prueba en Postman** (`api/.postman/`).

## Índice

1. Objetivo y alcance
2. Reglas de trabajo
3. Estructura del proyecto `api/`
4. Dependencias
5. Estrategia de tests
6. Carpeta `.postman`
7. Etapas y commits
8. Matriz de cobertura de requerimientos
9. Riesgos y planes B

---

## 1. Objetivo y alcance

Construir una API REST con **FastAPI + SQLAlchemy 2.0 + Alembic + PostgreSQL** que concentre toda la lógica de negocio de la tienda: usuarios, catálogo, precios y ofertas, carrito, pedidos, pagos con Mercado Pago, envíos, comprobantes y facturación en ARCA, contenido, reportes y SEO. El frontend nunca habla directo con la base ni con servicios externos.

### Relación etapas ↔ plan de 10 días

| Etapa | Tema | Día del plan general |
| --- | --- | --- |
| 0 | Esqueleto e infraestructura | Día 1 |
| 1 | Modelo de datos y seed | Día 1 |
| 2 | Autenticación y usuarios | Día 3 (se adelanta: el ABM del catálogo necesita el rol admin) |
| 3 | Catálogo | Día 2 |
| 4 | Precios y ofertas | Días 4 y 7 |
| 5 | Carrito | Día 4 |
| 6 | Pedidos | Día 4 |
| 7 | Pagos | Día 5 |
| 8 | Envíos | Día 6 |
| 9 | Comprobantes y ARCA | Día 8 |
| 10 | Contenido, reportes y SEO | Días 7 y 9 |
| 11 | Cierre y despliegue | Día 10 |

> Las etapas 2 y 3 pueden trabajarse en paralelo por dos personas: la etapa 3 solo necesita la dependencia `require_admin` del commit 10.

---

## 2. Reglas de trabajo

- **Ramas**: una rama `feature/<etapa>-<tema>` por etapa o sub-tema (ej. `feature/e7-webhook-mp`), con Pull Request a `develop`.
- **Formato de commit**: `feat:`, `fix:`, `chore:`, `docs:`, `test:` (se agrega `test:` para commits solo de tests).
- **Límite de tamaño**: cada commit tiene **como máximo 500 líneas cambiadas** (agregadas + borradas), medido con:
  ```bash
  git diff --cached --shortstat
  ```
  El límite incluye código, tests, migraciones y JSON de Postman. Si un commit lo supera, se divide (por ejemplo, separar los tests en un commit `test:` inmediatamente posterior).
- **Cada commit deja el proyecto en verde**: `ruff check .`, `ruff format --check .` y `pytest` pasan.
- **Tests junto con el código**: cada commit `feat:` trae sus tests; si no entran en el límite, el commit siguiente es el `test:` correspondiente.
- **Migraciones**: todo cambio en la base va con su migración Alembic **en el mismo commit** que el modelo. Nunca se toca la base a mano.
- **Postman**: cada etapa cierra con un commit `chore(postman):` que agrega su colección.
- **Secretos**: nunca en el repo. Solo `.env.example` con placeholders; los valores reales van en `.env` local (ignorado) y en Railway.

---

## 3. Estructura del proyecto `api/`

```
api/
├── app/
│   ├── main.py                 # crea la app, CORS, routers, lifespan (scheduler)
│   ├── core/
│   │   ├── config.py           # Settings (pydantic-settings)
│   │   ├── db.py               # engine, SessionLocal, get_db
│   │   ├── security.py         # hash de contraseñas, JWT, cookies
│   │   ├── deps.py             # get_current_user, require_admin, paginación
│   │   └── errors.py           # excepciones de dominio y handlers
│   ├── models/                 # SQLAlchemy: user, catalog, promotion, cart, order,
│   │                           # payment, shipping, invoice, content, audit
│   ├── schemas/                # Pydantic de entrada/salida por módulo
│   ├── routers/                # health, auth, users, categories, products, promotions,
│   │                           # prices, cart, orders, payments, shipping, invoices,
│   │                           # content, reports, seo, audit
│   ├── services/               # lógica de negocio: pricing, order_state, stock, cart,
│   │                           # orders, audit, reports, invoices, exports
│   ├── integrations/
│   │   ├── mercadopago.py
│   │   ├── arca/               # wsaa.py, wsfe.py, qr.py
│   │   ├── shipping/           # base.py, andreani.py, correo_ar.py, rate_table.py
│   │   ├── email.py            # interfaz + Resend + FakeEmail
│   │   └── storage.py          # interfaz + Cloudinary/R2 + FakeStorage
│   └── tasks/
│       └── reservations.py     # expiración de reservas de stock
├── alembic/
│   └── versions/
├── alembic.ini
├── scripts/
│   └── seed.py
├── tests/
│   ├── conftest.py
│   ├── factories.py
│   ├── unit/                   # lógica pura, sin base de datos
│   └── api/                    # endpoints con TestClient + Postgres de test
├── .postman/
│   ├── environments/
│   └── collections/
├── requirements.txt
├── requirements-dev.txt
├── pyproject.toml              # config de ruff y pytest
└── .env.example
```

**Criterio de capas**: los `routers/` solo validan entrada (Pydantic), llaman a un `service` y serializan la salida. Toda regla de negocio vive en `services/` para poder testearla de forma unitaria. Las llamadas a terceros viven en `integrations/` detrás de una interfaz, para reemplazarlas por *fakes* en los tests.

---

## 4. Dependencias

**`requirements.txt`**

| Paquete | Uso |
| --- | --- |
| `fastapi`, `uvicorn[standard]` | API y servidor |
| `sqlalchemy>=2.0`, `alembic`, `psycopg[binary]` | ORM, migraciones, driver PostgreSQL |
| `pydantic-settings`, `email-validator` | Configuración y validación de emails |
| `pwdlib[argon2]` | Hash de contraseñas con argon2 |
| `pyjwt` | Tokens JWT |
| `slowapi` | Límite de intentos de login |
| `mercadopago` | SDK oficial de Mercado Pago |
| `zeep`, `cryptography` | Cliente SOAP de ARCA y firma CMS del WSAA |
| `reportlab`, `qrcode` | PDFs de comprobantes y QR de ARCA |
| `openpyxl` | Exportación a Excel |
| `httpx` | Cliente HTTP para APIs de envíos |
| `resend` | Emails transaccionales |
| `cloudinary` | Subida de imágenes (o `boto3` si se elige R2) |
| `apscheduler` | Tarea programada de expiración de reservas |

**`requirements-dev.txt`**: `pytest`, `pytest-cov`, `ruff`, `respx` (mock de `httpx`), `time-machine` (control del reloj en tests de vencimientos).

---

## 5. Estrategia de tests

### Tipos de test

| Tipo | Carpeta | Qué prueba | Base de datos |
| --- | --- | --- | --- |
| **Unitario** | `tests/unit/` | Funciones puras de `services/`, `core/` e `integrations/` | No |
| **API** | `tests/api/` | Endpoints completos con `TestClient` | Postgres de test |

### Infraestructura (`tests/conftest.py`)

- Base `r40_test` en el Postgres de `docker-compose` (en CI, un *service container* de Postgres).
- Se crean las tablas con `alembic upgrade head` una vez por sesión de tests.
- Cada test corre dentro de una **transacción que se revierte** al terminar → tests aislados y rápidos.
- Fixtures: `db`, `client`, `client_cliente` (logueado como cliente), `client_admin` (logueado como admin).
- `tests/factories.py`: funciones para crear usuarios, categorías, productos, variantes, ofertas y pedidos de prueba.
- Las integraciones externas se reemplazan con *fakes* vía `app.dependency_overrides`: `FakeEmail` (guarda los emails enviados en una lista), `FakeStorage`, `FakeMercadoPago`, `FakeWsaa`/`FakeWsfe`. Las APIs HTTP de envíos se simulan con `respx`.

### Tests unitarios obligatorios

| Módulo | Casos principales |
| --- | --- |
| `core/config.py` | `DATABASE_URL` con `postgres://` y `postgresql://` se convierte a `postgresql+psycopg://` |
| `core/security.py` | Hash y verificación de contraseña; JWT válido, vencido y con firma inválida |
| `services/pricing.py` ⭐ | Sin oferta; oferta porcentaje; oferta monto fijo; oferta por producto, variante y categoría; dos ofertas vigentes → gana la de mayor descuento; ofertas no acumulables; oferta fuera de fecha (antes del inicio, después del fin, límites exactos); descuento mayor al precio → precio mínimo 0; redondeo con `Decimal` a 2 decimales |
| `services/order_state.py` ⭐ | Todas las transiciones válidas; todas las inválidas lanzan error (ej. `enviado → pagado`); `cancelado` solo desde `pendiente` o `pagado` |
| `integrations/mercadopago.py` ⭐ | Firma `x-signature` válida, inválida, con `ts` faltante, con header mal formado |
| `services/invoices.py` | Tipo de comprobante: monotributista → C; responsable inscripto + cliente RI → A; responsable inscripto + consumidor final → B |
| `integrations/arca/qr.py` | El JSON del QR tiene todos los campos exigidos y la URL se arma en base64 correctamente |
| `integrations/shipping/rate_table.py` | CP dentro de rango, fuera de rango, en el límite, CP inválido |
| `services/stock.py` | Reserva, liberación y cálculo de stock disponible (stock − reservas vigentes) |

⭐ = mínimos exigidos por el enunciado (precios y ofertas, flujo de estados, webhook).

### Cobertura y CI

- `pytest --cov=app --cov-report=term-missing` en CI.
- Objetivo: **≥ 80 % en `app/services/`** y 100 % en `pricing.py` y `order_state.py`.
- El CI (GitHub Actions) corre en cada Pull Request: `ruff check`, `ruff format --check`, `pytest --cov`.

---

## 6. Carpeta `.postman`

```
api/.postman/
├── environments/
│   ├── local.postman_environment.json      # base_url = http://localhost:8000
│   └── staging.postman_environment.json    # base_url = https://api-staging.<dominio>
└── collections/
    ├── 00-health.postman_collection.json
    ├── 01-auth.postman_collection.json
    ├── 02-catalogo.postman_collection.json
    ├── 03-precios-ofertas.postman_collection.json
    ├── 04-carrito.postman_collection.json
    ├── 05-pedidos.postman_collection.json
    ├── 06-pagos.postman_collection.json
    ├── 07-envios.postman_collection.json
    ├── 08-comprobantes.postman_collection.json
    ├── 09-contenido-reportes-seo.postman_collection.json
    └── 99-flujo-completo.postman_collection.json
```

### Convenciones

- Formato **Postman Collection v2.1**. Una colección por módulo para que cada commit sea chico y fácil de revisar.
- Variables de entorno: `base_url`, `admin_email`, `admin_password`, `cliente_email`, `cliente_password`, `mp_webhook_secret` (placeholder, cada uno carga el suyo localmente).
- Variables encadenadas que los requests guardan con `pm.collectionVariables.set(...)`: `category_id`, `product_id`, `variant_id`, `promotion_id`, `order_id`, `payment_id`, `reset_token`.
- Cada request tiene al menos un script de test:
  ```js
  pm.test("status 200", () => pm.response.to.have.status(200));
  pm.test("devuelve id", () => pm.expect(pm.response.json()).to.have.property("id"));
  ```
- Se incluyen también **casos negativos** (401 sin sesión, 403 cliente en ruta admin, 422 datos inválidos, 409 sin stock).
- El login guarda la cookie `httpOnly` en el *cookie jar* de Postman, así los requests siguientes quedan autenticados.
- El webhook de Mercado Pago calcula la firma HMAC en un **pre-request script** con `CryptoJS`, para simular notificaciones reales.
- **No se guardan secretos** en los environments commiteados: solo placeholders.

### Ejecución por consola (Newman)

```bash
npx newman run api/.postman/collections/01-auth.postman_collection.json \
  -e api/.postman/environments/local.postman_environment.json
```

---

## 7. Etapas y commits

Cada commit indica **archivos principales**, **contenido**, **tests** y una **estimación de líneas** (siempre ≤ 500). Cada etapa cierra con su **entregable** verificable.

---

### Etapa 0 — Esqueleto e infraestructura

**Commit 1 — `chore: estructura inicial de api/ y entorno local`** (~150 líneas)
- [ ] `api/` con carpetas `app/core`, `models`, `schemas`, `routers`, `services`, `integrations`, `tasks`, `tests`.
- [ ] `requirements.txt`, `requirements-dev.txt`, `pyproject.toml` (reglas de `ruff`, `testpaths`, `addopts` de pytest).
- [ ] `.env.example` con todas las variables: `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRE_MINUTES`, `FRONTEND_URL`, `CORS_ORIGINS`, `MP_ACCESS_TOKEN`, `MP_WEBHOOK_SECRET`, `RESEND_API_KEY`, `CLOUDINARY_URL`, `ARCA_CUIT`, `ARCA_CERT_PATH`, `ARCA_KEY_PATH`, `ARCA_ENV`, `ANDREANI_*`, `CORREO_AR_*`.
- [ ] `docker-compose.yml` en la raíz con PostgreSQL 16 y volumen persistente (bases `r40` y `r40_test`).

**Commit 2 — `feat: configuración, conexión a la base y endpoint /health`** (~250 líneas)
- [ ] `core/config.py`: clase `Settings` con `pydantic-settings`; validador que convierte `postgres://` / `postgresql://` a `postgresql+psycopg://`.
- [ ] `core/db.py`: `engine`, `SessionLocal`, dependencia `get_db`.
- [ ] `core/errors.py`: `DomainError` (con `code` y `message`) y handler que responde `{"error": {"code", "message", "details"}}`.
- [ ] `main.py`: app con CORS restringido a `CORS_ORIGINS`, `/docs` habilitado según entorno.
- [ ] `routers/health.py`: `GET /health` → `{"status": "ok", "db": "ok"}` (hace `SELECT 1`).
- [ ] Tests: `unit/test_config.py` (conversión de URL), `api/test_health.py`.

**Commit 3 — `feat: base de modelos con auditoría e inicialización de Alembic`** (~150 líneas)
- [ ] `models/base.py`: `Base` declarativa y `AuditMixin` con `created_at`, `updated_at`, `created_by`, `updated_by` (FK a `users`, nullable).
- [ ] Mecanismo para completar `created_by`/`updated_by` con el usuario del request (contexto por request + evento `before_flush`).
- [ ] `alembic init`, `env.py` leyendo `Settings.database_url` y `Base.metadata`; convención de nombres de constraints.

**Commit 4 — `chore: infraestructura de tests y CI`** (~180 líneas)
- [ ] `tests/conftest.py`: migración de la base de test, sesión con rollback por test, `client`, overrides de dependencias.
- [ ] `.github/workflows/api.yml`: Python 3.12, service Postgres, `pip install`, `ruff check`, `ruff format --check`, `pytest --cov`.

**Commit 5 — `chore(postman): environments y colección 00-health`** (~100 líneas)
- [ ] `environments/local` y `environments/staging`.
- [ ] `00-health`: `GET /health` con test de status y `db == "ok"`.

**✅ Entregable etapa 0:** `uvicorn app.main:app --reload` levanta, `/health` responde OK contra Postgres local, `/docs` visible, CI en verde.

---

### Etapa 1 — Modelo de datos y seed

**Commit 6 — `feat: modelos de usuarios y catálogo + migración 0001`** (~400 líneas)
- [ ] `users`: email único, `password_hash`, nombre, teléfono, `dni_cuit`, `condicion_iva` (consumidor final, RI, monotributo, exento), `role` (cliente/admin), `is_active`.
- [ ] `addresses`: usuario, calle, número, piso/depto, ciudad, provincia, CP, `is_default`.
- [ ] `categories`: nombre, slug único, descripción, estado (`activa`, `oculta`, `proximamente`), orden.
- [ ] `products`: nombre, slug único, descripción, categoría, estado (`activo`, `oculto`, `proximamente`), `is_featured`.
- [ ] `product_variants`: producto, SKU único, talle, color, `price` (`Numeric(12,2)`), `stock`, `is_active`.
- [ ] `product_images`: producto, variante (opcional), URL, `public_id` del storage, orden.
- [ ] Índices: `products.slug`, `products.category_id`, `product_variants.sku`; índice para búsqueda por texto (`pg_trgm` sobre `name`).
- [ ] Migración `0001_usuarios_catalogo`.

**Commit 7 — `feat: modelos de precios, carrito, pedidos, pagos y envíos + migración 0002`** (~450 líneas)
- [ ] `promotions`: nombre, `type` (`percentage`/`fixed`), `value`, `scope` (`product`/`variant`/`category`), `target_id`, `starts_at`, `ends_at`, `is_active`.
- [ ] `price_history`: variante, precio anterior, precio nuevo, usuario, fecha, motivo (individual/masivo).
- [ ] `carts` (uno por usuario) y `cart_items` (variante, cantidad).
- [ ] `orders`: número legible, usuario (nullable), datos del comprador (nombre, email, teléfono, DNI/CUIT, condición IVA), estado, subtotal, descuento, costo de envío, total, método de pago, método de envío, dirección (copia JSON), `reserved_until`.
- [ ] `order_items`: copia de nombre de producto, descripción de variante, SKU, precio unitario, precio con descuento, cantidad, subtotal.
- [ ] `payments`: pedido, medio (`mercadopago`/`transferencia`), `mp_preference_id`, `mp_payment_id` (**único**), estado, monto, `raw_notifications` (JSONB).
- [ ] `shipments`: pedido, empresa, costo, número de seguimiento, estado.
- [ ] Migración `0002_ventas`.

**Commit 8 — `feat: modelos de facturación, contenido y auditoría + migración 0003`** (~350 líneas)
- [ ] `invoices`: pedido, tipo (`comprobante_interno`, `A`, `B`, `C`), punto de venta, número, CAE, vencimiento del CAE, URL/ruta del PDF, estado, respuesta de ARCA (JSONB).
- [ ] `shipping_rates`: zona, CP desde, CP hasta, precio, plazo estimado, `is_active`.
- [ ] `faq_items`: pregunta, respuesta, orden, `is_active`.
- [ ] `settings`: `key` única, `value` (JSONB) → WhatsApp, datos bancarios, "Quiénes somos", condición fiscal de R40, dirección de retiro en local.
- [ ] `audit_log`: acción, entidad, `entity_id`, usuario, fecha, `detail` (JSONB).
- [ ] Migración `0003_facturacion_contenido`.

**Commit 9 — `feat: script de seed`** (~250 líneas)
- [ ] `scripts/seed.py`: categorías y productos extraídos del `index.html` actual (datos en un `seed_data.json`), categoría **"Ropa y equipaje"** con estado `proximamente`, settings iniciales, usuario admin tomado de variables de entorno.
- [ ] El seed es **idempotente** (se puede correr dos veces sin duplicar, usa *upsert* por slug/SKU).
- [ ] Tests: `api/test_seed.py` (correr dos veces → mismas cantidades; "Ropa y equipaje" queda `proximamente`).

**✅ Entregable etapa 1:** `alembic upgrade head` crea las 18 tablas; `alembic downgrade base` las elimina sin errores; el seed carga los productos reales.

---

### Etapa 2 — Autenticación y usuarios

**Commit 10 — `feat: seguridad base (hash, JWT, cookies y dependencias de rol)`** (~300 líneas)
- [ ] `core/security.py`: `hash_password`, `verify_password` (argon2), `create_access_token`, `decode_token`, `set_auth_cookie` (`httpOnly`, `Secure` en staging/prod, `SameSite=Lax`), `clear_auth_cookie`.
- [ ] `core/deps.py`: `get_current_user` (lee la cookie, valida JWT, verifica que el usuario siga activo), `get_optional_user`, `require_admin` (403 si no es admin).
- [ ] Tests unitarios: hash/verify, token válido/vencido/manipulado; tests de dependencias con una ruta de prueba.

**Commit 11 — `feat: registro, login, logout y sesión actual`** (~450 líneas)
- [ ] `POST /auth/register` (email único, contraseña mínima de 8 caracteres), `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`.
- [ ] `POST /admin/auth/login`: mismo flujo pero rechaza usuarios no admin (lo usa `/admin/login` del frontend).
- [ ] Usuario con `is_active = false` → 403 "Usuario desactivado".
- [ ] Límite de intentos con `slowapi` (ej. 5 por minuto por IP + email) → 429.
- [ ] Mensaje de error genérico en login ("Email o contraseña incorrectos") para no revelar qué emails existen.
- [ ] Tests API: registro OK, email duplicado (409), login OK setea cookie, contraseña incorrecta (401), usuario desactivado (403), 6.º intento (429), `/auth/me` sin cookie (401), logout borra la cookie.

**Commit 12 — `feat: recuperación de contraseña y servicio de email`** (~400 líneas)
- [ ] `integrations/email.py`: interfaz `EmailSender`, implementación `ResendEmailSender` y `FakeEmailSender` para tests/local; plantillas simples en `app/templates/email/`.
- [ ] Tabla `password_reset_tokens` (usuario, hash del token, vencimiento, `used_at`) + migración `0004`.
- [ ] `POST /auth/forgot-password`: genera token aleatorio, guarda **solo su hash**, envía link `FRONTEND_URL/reset-password?token=...` con vencimiento de 1 hora. Siempre responde 200 (no revela si el email existe).
- [ ] `POST /auth/reset-password`: valida token, vencimiento y que no se haya usado; cambia la contraseña.
- [ ] Tests: email enviado (FakeEmail), token vencido (con `time-machine`), token reutilizado, token inválido.

**Commit 13 — `feat: perfil, direcciones y gestión de usuarios por el admin`** (~400 líneas)
- [ ] `GET/PATCH /users/me` (nombre, teléfono, DNI/CUIT, condición IVA; no permite cambiar rol ni email sin validación).
- [ ] `GET/POST/PATCH/DELETE /users/me/addresses` (una dirección por defecto).
- [ ] `POST /admin/users` (**solo un admin crea admins**), `GET /admin/users` (paginado, filtro por rol), `PATCH /admin/users/{id}` (activar/desactivar).
- [ ] Tests: cliente no puede crear admin (403), registro público no puede enviar `role=admin` (se ignora), CRUD de direcciones, un usuario no ve direcciones de otro (404).

**Commit 14 — `chore(postman): colección 01-auth`** (~300 líneas)
- [ ] Registro, login cliente, me, editar perfil, CRUD direcciones, forgot/reset, logout.
- [ ] Login admin, crear admin, desactivar usuario, login de usuario desactivado (403), cliente en ruta admin (403).

**✅ Entregable etapa 2:** clientes y administrador inician sesión; las rutas `/admin/*` devuelven 403 a clientes; la recuperación de contraseña funciona con email real en staging.

---

### Etapa 3 — Catálogo

**Commit 15 — `feat: ABM de categorías`** (~300 líneas)
- [ ] `GET /categories` (público: solo `activa` y `proximamente`), `GET /categories/{slug}`.
- [ ] `POST/PATCH/DELETE /admin/categories` (borrado lógico si tiene productos).
- [ ] Generación de slug con `services/slug.py` (normaliza acentos y ñ, agrega sufijo si existe).
- [ ] Tests unitarios del slug ("Ropa y Equipaje" → `ropa-y-equipaje`, duplicado → `-2`) y tests API del ABM.

**Commit 16 — `feat: ABM de productos`** (~400 líneas)
- [ ] `POST/PATCH/DELETE /admin/products`, `GET /admin/products` (incluye ocultos).
- [ ] Estados `activo`, `oculto`, `proximamente`; un producto de categoría `proximamente` hereda "no comprable".
- [ ] Campo calculado `is_purchasable` en la respuesta.
- [ ] Tests: crear, editar, slug único, producto oculto no aparece en el público, producto de "Ropa y equipaje" con `is_purchasable = false`.

**Commit 17 — `feat: ABM de variantes`** (~350 líneas)
- [ ] `POST/PATCH/DELETE /admin/products/{id}/variants`; SKU único; precio > 0; stock ≥ 0.
- [ ] Un producto sin variantes explícitas tiene una variante "única" (simplifica carrito y pedidos).
- [ ] Tests: SKU duplicado (409), precio negativo (422), stock negativo (422).

**Commit 18 — `feat: listado público de productos con búsqueda y filtros`** (~400 líneas)
- [ ] `GET /products?page=&size=&q=&category=&min_price=&max_price=&sort=` → `{items, total, page, size, pages}`.
- [ ] Búsqueda por texto en nombre y descripción (`ILIKE` + `pg_trgm`); filtro por precio sobre el **precio final con oferta**; orden por precio y novedades.
- [ ] `GET /products/{slug}`: detalle con variantes, imágenes, precio original, precio final y oferta aplicada.
- [ ] `core/deps.py`: dependencia reutilizable de paginación (tamaño máximo 100).
- [ ] Tests: paginación, búsqueda, cada filtro, combinación de filtros, slug inexistente (404).

**Commit 19 — `feat: imágenes de productos`** (~300 líneas)
- [ ] `integrations/storage.py`: interfaz `ImageStorage` con `upload`/`delete`; `CloudinaryStorage` y `FakeStorage`.
- [ ] `POST /admin/products/{id}/images` (multipart; valida tipo `jpg/png/webp` y tamaño máx. 5 MB), `PATCH` (orden, variante), `DELETE`.
- [ ] Tests con `FakeStorage`: subida, tipo inválido (422), reordenar, borrar.

**Commit 20 — `chore(postman): colección 02-catalogo`** (~350 líneas)
- [ ] ABM admin de categorías, productos, variantes e imágenes; listado público con cada filtro; detalle por slug; casos 404/409/422.

**✅ Entregable etapa 3:** catálogo navegable desde `/docs` con los productos reales; "Ropa y equipaje" visible pero no comprable.

---

### Etapa 4 — Precios y ofertas

**Commit 21 — `feat: servicio de cálculo de precios`** (~400 líneas)
- [ ] `services/pricing.py`, función pura: `calculate_price(variant, product, category, promotions, now) -> PriceResult(original, final, discount, promotion_id)`.
- [ ] Reglas: solo ofertas activas y vigentes (`starts_at ≤ now < ends_at`), alcance variante/producto/categoría, **se aplica la de mayor descuento sin acumular**, precio final mínimo 0, `Decimal` con redondeo `ROUND_HALF_UP` a 2 decimales.
- [ ] Función `get_active_promotions(db, variants, now)` que trae las ofertas en una sola consulta (evita N+1 en listados).
- [ ] **Tests unitarios exhaustivos** (ver tabla de la sección 5): es el corazón del requerimiento "el precio siempre se calcula en la API".

**Commit 22 — `feat: ABM de ofertas`** (~350 líneas)
- [ ] `GET/POST/PATCH/DELETE /admin/promotions`; filtros por vigentes/futuras/vencidas.
- [ ] Validaciones: `ends_at > starts_at`; porcentaje entre 1 y 100; monto fijo > 0; `target_id` existe según el alcance.
- [ ] `GET /promotions/active` (público, para la home).
- [ ] Tests: cada validación (422) y que el listado público refleja la oferta.

**Commit 23 — `feat: edición de precios individual y masiva con historial y auditoría`** (~400 líneas)
- [ ] `services/audit.py`: `log(db, action, entity, entity_id, user, detail)` reutilizable por todos los módulos.
- [ ] `PATCH /admin/variants/{id}/price`: registra en `price_history` y `audit_log`.
- [ ] `POST /admin/prices/bulk`: `{category_id | product_ids, mode: "percentage" | "fixed", value, dry_run}`. Con `dry_run=true` devuelve la vista previa sin guardar. Todo en una transacción.
- [ ] `GET /admin/prices/history?variant_id=&from=&to=` paginado.
- [ ] Tests: aumento de 10 % a una categoría (verifica cada variante e historial), `dry_run` no modifica, valores que dejarían precio ≤ 0 se rechazan.

**Commit 24 — `chore(postman): colección 03-precios-ofertas`** (~250 líneas)
- [ ] Crear oferta porcentaje y monto fijo, ver precio final en el detalle, dos ofertas superpuestas (gana la mayor), aumento masivo con `dry_run` y real, historial.

**✅ Entregable etapa 4:** el admin cambia precios (individual y masivo) y crea ofertas con vigencia; el catálogo muestra precio original y final correctos.

---

### Etapa 5 — Carrito

**Commit 25 — `feat: carrito en servidor para clientes`** (~400 líneas)
- [ ] `GET /cart`, `POST /cart/items`, `PATCH /cart/items/{id}`, `DELETE /cart/items/{id}`, `DELETE /cart`.
- [ ] `services/cart.py`: valida que la variante exista, sea comprable y tenga stock disponible (stock − reservas vigentes).
- [ ] La respuesta siempre trae precios **recalculados** con `pricing.py`, subtotal y total.
- [ ] Tests: agregar, sumar cantidad del mismo ítem, cantidad > stock (409 con detalle del stock disponible), producto `proximamente` (409), carrito de un usuario no visible para otro.

**Commit 26 — `feat: fusión de carrito y cotización de carrito de invitado`** (~350 líneas)
- [ ] `POST /cart/merge`: recibe los ítems del navegador `[{variant_id, quantity}]` y los fusiona con el del servidor (suma cantidades hasta el stock disponible; informa ajustes).
- [ ] `POST /cart/quote` (público): recibe ítems del invitado, devuelve precios actuales y **avisos**: `price_changed` (con precio anterior y nuevo si el front envía el que mostró), `out_of_stock`, `quantity_adjusted`, `not_purchasable`. **Nunca se usa el precio enviado por el front para cobrar**: solo se usa para comparar y avisar.
- [ ] Tests: fusión con ítems repetidos, fusión que supera stock, cotización con precio cambiado y producto sin stock.

**Commit 27 — `chore(postman): colección 04-carrito`** (~200 líneas)
- [ ] Carrito de cliente completo, cotización de invitado, fusión al loguearse, casos de stock insuficiente.

**✅ Entregable etapa 5:** carrito de cliente persistente y cotización de carrito de invitado con avisos de precio y stock.

---

### Etapa 6 — Pedidos

**Commit 28 — `feat: máquina de estados del pedido`** (~250 líneas)
- [ ] `services/order_state.py`: enum `OrderStatus` y tabla de transiciones:

  | Desde | Hacia permitido |
  | --- | --- |
  | `pendiente_pago` | `pagado`, `cancelado` |
  | `pagado` | `en_preparacion`, `cancelado` |
  | `en_preparacion` | `enviado` |
  | `enviado` | `entregado` |
  | `entregado`, `cancelado` | — (estados finales) |

- [ ] `transition(order, new_status, user, reason)`: valida, cambia el estado, registra en `audit_log` y dispara la notificación por email (inyectada, para poder testear).
- [ ] **Tests unitarios**: todas las transiciones válidas y todas las inválidas (parametrizado).

**Commit 29 — `feat: creación de pedidos con reserva de stock`** (~450 líneas)
- [ ] `POST /orders`: con sesión (toma datos del usuario) o como **invitado** (exige nombre, email, teléfono y DNI/CUIT validados con Pydantic).
- [ ] Recibe solo `variant_id` y `quantity` por ítem, método de envío y dirección; **recalcula precios** en la API.
- [ ] `services/stock.py`: bloquea las variantes con `SELECT ... FOR UPDATE`, verifica stock disponible y crea la reserva (`reserved_until = now + 30 min`) en la misma transacción → evita sobreventa con compras simultáneas.
- [ ] `order_items` guarda **copia** de nombre, variante, SKU y precios.
- [ ] Al crear desde un carrito de servidor, el carrito se vacía.
- [ ] Tests: pedido de cliente, pedido de invitado, invitado sin DNI (422), stock insuficiente (409), cambio de precio posterior no altera el pedido, número de pedido único.

**Commit 30 — `feat: expiración de reservas y notificaciones de estado`** (~350 líneas)
- [ ] `tasks/reservations.py`: `expire_unpaid_orders(db, now)` cancela pedidos `pendiente_pago` con `reserved_until < now` (y sin transferencia informada), libera stock y registra auditoría.
- [ ] Registro en `APScheduler` dentro del `lifespan` de la app (cada 1 minuto). Alternativa documentada: endpoint protegido `POST /internal/tasks/expire-reservations` para un cron de Railway.
- [ ] Emails por cada cambio de estado (plantillas por estado).
- [ ] Tests con `time-machine`: a los 29 min sigue reservado, a los 31 min se cancela y el stock vuelve; se envía el email de cancelación.

**Commit 31 — `feat: consulta y gestión de pedidos`** (~400 líneas)
- [ ] Cliente: `GET /orders/me` (paginado), `GET /orders/me/{id}` (estado, ítems, seguimiento, comprobantes).
- [ ] Invitado: `GET /orders/lookup?number=&email=` para consultar su pedido.
- [ ] Admin: `GET /admin/orders?status=&from=&to=&q=&page=`, `GET /admin/orders/{id}`, `POST /admin/orders/{id}/status` (usa `order_state.transition`; al cancelar un pedido pagado libera stock).
- [ ] Tests: filtros, cliente no ve pedidos ajenos (404), transición inválida (409), email enviado al cambiar estado.

**Commit 32 — `chore(postman): colección 05-pedidos`** (~250 líneas)
- [ ] Pedido de cliente y de invitado, mis pedidos, lookup de invitado, admin: listado con filtros y cambio de estado válido e inválido.

**✅ Entregable etapa 6:** se arma un carrito y se genera un pedido `pendiente_pago` con stock reservado; los impagos se cancelan solos a los 30 minutos.

---

### Etapa 7 — Pagos

**Commit 33 — `feat: preferencia de pago de Mercado Pago`** (~300 líneas)
- [ ] `integrations/mercadopago.py`: interfaz `PaymentGateway` con `create_preference(order)` y `get_payment(payment_id)`; implementación con el SDK `mercadopago` y `FakeMercadoPago`.
- [ ] `POST /orders/{id}/payments/mercadopago`: crea la preferencia con ítems del pedido, `external_reference = order.id`, `back_urls` (aprobado/pendiente/rechazado del frontend), `notification_url` (webhook), `expiration_date_to` igual al vencimiento de la reserva. Devuelve `init_point`.
- [ ] Solo para pedidos `pendiente_pago`; guarda `mp_preference_id` en `payments`.
- [ ] Tests con fake: preferencia creada, pedido ya pagado (409), pedido de otro usuario (404).

**Commit 34 — `feat: webhook de Mercado Pago con firma e idempotencia`** (~350 líneas)
- [ ] `POST /payments/webhook/mercadopago`.
- [ ] Verificación de firma: parsea `x-signature` (`ts=...,v1=...`) y `x-request-id`; arma el manifiesto `id:{data.id};request-id:{x-request-id};ts:{ts};`; calcula HMAC-SHA256 con `MP_WEBHOOK_SECRET` y compara con `hmac.compare_digest`. Firma inválida → 401.
- [ ] Consulta el pago real con `get_payment` (no confía en el cuerpo de la notificación), toma `external_reference` y monto.
- [ ] **Idempotencia**: guarda la notificación en `raw_notifications`; `mp_payment_id` es único; si el pago ya fue procesado con el mismo estado, responde 200 sin hacer nada.
- [ ] `approved` + monto igual al total → `transition(pagado)`; `rejected`/`cancelled` → registra sin cambiar el pedido; monto distinto → registra alerta en `audit_log` y no marca como pagado.
- [ ] El estado **solo** cambia por este webhook; las `back_urls` no modifican nada.
- [ ] Unit tests de la verificación de firma.

**Commit 35 — `test: escenarios del webhook de pagos`** (~300 líneas)
- [ ] Firma válida → pedido pagado y email de confirmación enviado.
- [ ] Firma inválida → 401 y pedido sin cambios.
- [ ] Misma notificación dos veces → un solo pago, un solo email.
- [ ] Pago aprobado con monto distinto → no se marca pagado.
- [ ] Pago rechazado → pedido sigue `pendiente_pago`.
- [ ] Notificación de pedido inexistente → 200 (para que MP no reintente) y registro en log.

**Commit 36 — `feat: pago por transferencia bancaria`** (~300 líneas)
- [ ] `POST /orders/{id}/payments/transfer`: marca el medio de pago y devuelve los datos bancarios desde `settings`; extiende la reserva (ej. 48 h, configurable) para que no se cancele a los 30 min.
- [ ] `POST /admin/orders/{id}/payments/confirm-transfer`: el admin confirma el pago manualmente → `transition(pagado)` + auditoría.
- [ ] Email de confirmación de compra (compartido con Mercado Pago).
- [ ] Tests: confirmación por admin, cliente no puede confirmar (403), confirmar dos veces (409).

**Commit 37 — `chore(postman): colección 06-pagos`** (~250 líneas)
- [ ] Crear preferencia, webhook simulado con firma calculada en pre-request (`CryptoJS.HmacSHA256`), webhook duplicado, firma inválida, transferencia y confirmación manual.

**✅ Entregable etapa 7:** compra completa con usuarios de prueba de Mercado Pago en staging; transferencia confirmada desde el admin.

---

### Etapa 8 — Envíos

**Commit 38 — `feat: tarifas por zona y retiro en local`** (~350 líneas)
- [ ] `GET/POST/PATCH/DELETE /admin/shipping-rates` (zona, rango de CP, precio, plazo); valida que los rangos no se superpongan.
- [ ] `integrations/shipping/rate_table.py`: busca la zona por CP.
- [ ] Retiro en local: opción de costo 0 si `settings.pickup_enabled` es verdadero, con la dirección configurada.
- [ ] Tests unitarios de resolución por CP y tests API del ABM (rango superpuesto → 409).

**Commit 39 — `feat: cotización de envíos con proveedores y fallback`** (~400 líneas)
- [ ] `integrations/shipping/base.py`: interfaz `ShippingProvider.quote(cp, items) -> list[ShippingOption]`.
- [ ] `andreani.py` y `correo_ar.py` con `httpx`, activos solo si hay credenciales en el entorno; timeout corto.
- [ ] `POST /shipping/quote`: `{cp, items}` → opciones de todos los proveedores disponibles; si fallan o no hay credenciales, usa la tabla por zona; siempre agrega retiro en local si aplica.
- [ ] La creación del pedido (commit 29) valida el método elegido y recalcula el costo en la API.
- [ ] Tests con `respx`: proveedor responde OK, proveedor da timeout → fallback a tabla, sin credenciales → tabla, CP sin zona → 422.

**Commit 40 — `feat: seguimiento de envíos`** (~250 líneas)
- [ ] `PATCH /admin/orders/{id}/shipment` (empresa, número de seguimiento); al cargar el número se sugiere pasar a `enviado`.
- [ ] El seguimiento aparece en `GET /orders/me/{id}` y en el email de "enviado".
- [ ] Tests: carga de seguimiento, cliente lo ve, email con el número.

**Commit 41 — `chore(postman): colección 07-envios`** (~200 líneas)
- [ ] ABM de tarifas, cotización por CP, retiro en local, carga de seguimiento.

**✅ Entregable etapa 8:** el checkout cotiza el envío por CP (por API o por tabla) y el pedido guarda método, costo y seguimiento.

---

### Etapa 9 — Comprobantes y facturación ARCA

**Commit 42 — `feat: comprobante de venta en PDF`** (~300 líneas)
- [ ] `services/pdf.py` con `reportlab`: datos de R40, comprador, ítems, envío, totales y número de pedido.
- [ ] Se genera para pedidos `pagado` o posteriores; se guarda en el storage y se registra en `invoices` con tipo `comprobante_interno`.
- [ ] `GET /orders/me/{id}/receipt` (cliente) y `GET /admin/orders/{id}/receipt` (admin).
- [ ] Tests: PDF generado (empieza con `%PDF`), pedido impago (409), cliente ajeno (404).

**Commit 43 — `feat: cliente WSAA de ARCA`** (~350 líneas)
- [ ] `integrations/arca/wsaa.py`: arma el TRA (XML con `uniqueId`, `generationTime`, `expirationTime`, `service=wsfe`), lo firma en CMS/PKCS#7 con `cryptography` usando certificado y clave, llama a `loginCms` con `zeep` (URL de homologación o producción según `ARCA_ENV`).
- [ ] Cachea `token` y `sign` hasta su vencimiento (12 h) en la tabla `settings` (clave `arca_ta`) para sobrevivir reinicios.
- [ ] Tests: TRA bien formado, reutilización del token vigente, renovación al vencer (con `time-machine`), cliente SOAP simulado.

**Commit 44 — `feat: cliente WSFEv1 y emisión de factura`** (~400 líneas)
- [ ] `integrations/arca/wsfe.py`: `FECompUltimoAutorizado` y `FECAESolicitar`.
- [ ] `services/invoices.py`: `determine_invoice_type(r40_condicion, cliente_condicion)` → C (11) si R40 es monotributista; A (1) si R40 es RI y el cliente es RI; B (6) en otro caso. Arma importes (neto, IVA, total) y documento del receptor (DNI 96 / CUIT 80 / consumidor final 99).
- [ ] `POST /admin/orders/{id}/invoice`: emite la factura, guarda CAE y vencimiento, registra en `audit_log`. Si ARCA rechaza, guarda observaciones y responde 502 con el detalle.
- [ ] No permite facturar dos veces el mismo pedido (409).

**Commit 45 — `test: facturación electrónica`** (~300 líneas)
- [ ] Unit tests de `determine_invoice_type` (todas las combinaciones) y del armado de importes.
- [ ] Tests API con `FakeWsfe`: CAE aprobado, rechazo con observaciones, doble emisión (409), numeración correlativa a partir del último autorizado.

**Commit 46 — `feat: PDF de factura con CAE y QR de ARCA`** (~350 líneas)
- [ ] `integrations/arca/qr.py`: JSON `{ver, fecha, cuit, ptoVta, tipoCmp, nroCmp, importe, moneda, ctz, tipoDocRec, nroDocRec, tipoCodAut: "E", codAut}` → base64 → `https://www.afip.gob.ar/fe/qr/?p=...` → imagen QR.
- [ ] PDF de factura con tipo, punto de venta, número, CAE, vencimiento del CAE y QR.
- [ ] `GET /orders/me/{id}/invoice` y `GET /admin/orders/{id}/invoice/pdf`.
- [ ] Tests unitarios del QR y test API de descarga.

**Commit 47 — `feat: exportación de ventas a CSV y Excel`** (~200 líneas)
- [ ] `GET /admin/exports/sales?from=&to=&format=csv|xlsx`: un renglón por pedido pagado (fecha, número, comprador, DNI/CUIT, condición IVA, subtotal, envío, total, medio de pago, CAE si tiene).
- [ ] Tests: CSV con cabeceras correctas, XLSX legible con `openpyxl`, rango sin ventas.

**Commit 48 — `chore(postman): colección 08-comprobantes`** (~200 líneas)
- [ ] Comprobante PDF, emisión de factura (homologación), descarga de factura, exportación CSV/XLSX.

**✅ Entregable etapa 9:** facturas emitidas en homologación desde el panel; comprobante interno y exportación disponibles como plan B.

---

### Etapa 10 — Contenido, reportes y SEO

**Commit 49 — `feat: preguntas frecuentes y configuración general`** (~350 líneas)
- [ ] `GET /faq` (público, ordenado), `POST/PATCH/DELETE /admin/faq`, `POST /admin/faq/reorder`.
- [ ] `GET /settings/public` (WhatsApp, "Quiénes somos", datos bancarios, retiro en local) y `GET/PATCH /admin/settings` con esquema Pydantic por clave (no se guarda cualquier JSON).
- [ ] Cambios de settings registrados en `audit_log`.
- [ ] Tests: orden de FAQ, settings con formato inválido (422), cliente no edita settings (403).

**Commit 50 — `feat: reportes de ventas`** (~450 líneas)
- [ ] `services/reports.py` con consultas agregadas en SQL:
  - `GET /admin/reports/sales?period=day|week|month&from=&to=&category=` → por período: total facturado, cantidad de pedidos, ticket promedio.
  - `GET /admin/reports/orders-by-status?from=&to=`.
  - `GET /admin/reports/top-products?from=&to=&category=&limit=`.
  - `GET /admin/reports/low-stock?threshold=&page=` (umbral configurable en settings).
  - `GET /admin/reports/dashboard`: resumen de hoy, semana y mes para el panel.
- [ ] Solo cuentan pedidos `pagado` o posteriores para ventas.
- [ ] Tests con datos armados por factories: totales y ticket promedio exactos, agrupación semanal, filtro por categoría, paginación de stock bajo.

**Commit 51 — `feat: sitemap.xml`** (~150 líneas)
- [ ] `GET /sitemap.xml` (`application/xml`): home, páginas institucionales, categorías activas/próximamente y productos activos con `lastmod` desde `updated_at`. URLs con el dominio del frontend (`FRONTEND_URL`).
- [ ] Tests: incluye productos activos, excluye ocultos, XML válido.

**Commit 52 — `chore(postman): colección 09-contenido-reportes-seo`** (~250 líneas)
- [ ] ABM de FAQ, settings públicos y admin, cada reporte con filtros, sitemap.

**✅ Entregable etapa 10:** el admin edita FAQ y configuración; el dashboard tiene todos sus datos; el sitemap lista el catálogo.

---

### Etapa 11 — Cierre y despliegue

**Commit 53 — `feat: consulta del registro de auditoría`** (~250 líneas)
- [ ] `GET /admin/audit-log?entity=&action=&user_id=&from=&to=&page=`.
- [ ] Verificación de que cambios de precio, cambios de estado y facturas emitidas siempre generan registro.
- [ ] Tests: cada acción auditable genera su entrada; filtros.

**Commit 54 — `chore: configuración de despliegue en Railway`** (~200 líneas)
- [ ] `api/railway.json`: start `uvicorn app.main:app --host 0.0.0.0 --port $PORT`, pre-deploy `alembic upgrade head`, healthcheck `/health`.
- [ ] Cookies `Secure` + `SameSite` según entorno; CORS solo para el dominio del frontend; `/docs` activo en staging.
- [ ] Logging estructurado (request id, ruta, status, duración) y log de errores no controlados.
- [ ] Headers de seguridad básicos (`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`).

**Commit 55 — `test: flujo completo de compra`** (~300 líneas)
- [ ] `tests/api/test_flujo_completo.py`: registro → carrito → cotización de envío → pedido → preferencia MP → webhook aprobado → comprobante → factura (fakes) → reportes reflejan la venta.
- [ ] Variante como invitado con transferencia confirmada por el admin.
- [ ] Variante de pedido impago que expira y libera stock.

**Commit 56 — `docs: README de la API y colección de flujo completo`** (~300 líneas)
- [ ] `api/README.md`: cómo levantar en local, variables de entorno, correr tests, correr migraciones y seed, usar Postman/Newman, probar Mercado Pago y ARCA en homologación.
- [ ] `99-flujo-completo`: la compra de punta a punta en orden, lista para el *Collection Runner* o Newman contra staging.

**✅ Entregable etapa 11:** API desplegada en staging con migraciones automáticas, documentación en `/docs`, flujo completo validado por tests y por Newman.

---

## 8. Matriz de cobertura de requerimientos

| Requerimiento de la Parte 1 | Commits | Tests | Postman |
| --- | --- | --- | --- |
| Registro, login, usuario desactivado | 10, 11 | unit + API | 01 |
| Recuperación de contraseña con vencimiento | 12 | API | 01 |
| Perfil y direcciones | 13 | API | 01 |
| Solo admin crea admins | 13 | API | 01 |
| ABM categorías, productos, variantes, imágenes | 15–19 | unit + API | 02 |
| "Ropa y equipaje" próximamente, no comprable | 9, 16 | API | 02 |
| Listado paginado, búsqueda y filtros | 18 | API | 02 |
| Precios individuales y masivos con historial | 23 | API | 03 |
| Ofertas, mejor descuento sin acumular | 21, 22 | **unit** + API | 03 |
| Carrito invitado/cliente y fusión | 25, 26 | API | 04 |
| Precio siempre calculado en la API, avisos | 21, 26, 29 | unit + API | 04 |
| Pedidos con o sin cuenta, copia de precios | 29 | API | 05 |
| Flujo de estados | 28, 31 | **unit** + API | 05 |
| Reserva de stock 30 min y cancelación | 29, 30 | API | 05 |
| Email por cambio de estado | 28, 30 | API | — |
| Mercado Pago: preferencia | 33 | API | 06 |
| Webhook con firma e idempotente | 34, 35 | **unit** + API | 06 |
| Transferencia con confirmación manual | 36 | API | 06 |
| Cotización por CP, tabla por zona, retiro en local | 38, 39 | unit + API | 07 |
| Número de seguimiento | 40 | API | 07 |
| Comprobante PDF | 42 | API | 08 |
| ARCA WSAA + WSFEv1, tipo de comprobante | 43–45 | unit + API | 08 |
| PDF de factura con CAE y QR | 46 | unit + API | 08 |
| Exportación CSV/Excel | 47 | API | 08 |
| FAQ y configuración general | 49 | API | 09 |
| Reportes | 50 | API | 09 |
| sitemap.xml | 51 | API | 09 |
| Seguridad: argon2, JWT httpOnly, rate limit, CORS | 2, 10, 11, 54 | unit + API | 01 |
| Validación Pydantic y errores claros | 2 y todos | API | casos 4xx |
| Trazabilidad y auditoría de registros | 3, 23, 28, 44, 53 | API | — |
| Migraciones Alembic | 3, 6–8, 12 | CI | — |
| Swagger en staging | 2, 54 | — | — |

---

## 9. Riesgos y planes B

| Riesgo | Plan B en el backend | Commits que lo soportan |
| --- | --- | --- |
| Andreani / Correo Argentino sin credenciales | Tabla de tarifas por zona + carga manual del seguimiento | 38, 40 |
| ARCA demora certificado u homologación | Comprobante PDF interno + exportación a Excel/CSV | 42, 47 |
| Mercado Pago sin credenciales de producción | Transferencia bancaria con confirmación manual | 36 |
| Condición fiscal de R40 sin definir | El tipo de comprobante se lee de `settings`; la regla ya contempla C, A y B | 44, 49 |
| Cloudinary vs. R2 sin definir | Interfaz `ImageStorage`; cambiar de proveedor es una implementación nueva | 19 |
