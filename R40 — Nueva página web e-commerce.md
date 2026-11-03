# R40 — Nueva página web e-commerce

Nov 2, 2026 · @Ignacio

## Reglas de organización

### Repositorio

- Todo el código vive en un único repositorio **privado** en GitHub llamado `r40-web` (monorepo).
- La raíz del repositorio debe contener:
  - `api/` con el proyecto Python (FastAPI), su `requirements.txt` y su `.env.example`.
  - `frontend/` con el proyecto React (Vite), su `package.json` y su `.env.example`.
  - `docker-compose.yml` solo para desarrollo local: levanta un PostgreSQL con volumen persistente para no depender de Railway al programar.
  - `README.md` con descripción del proyecto, integrantes y pasos para levantarlo en local.
- Ningún secreto (credenciales de Mercado Pago, ARCA, Andreani, JWT) se sube al repositorio. Se cargan como variables de entorno en Railway y en un `.env` local ignorado por git.

### Ramas y entornos

| Rama | Entorno en Railway | Uso |
| --- | --- | --- |
| `main` | Producción | Solo recibe merges desde `develop` ya probados |
| `develop` | Staging | Integración diaria; cada push se despliega solo |
| `feature/<nombre>` | Ninguno | Una rama por tarea; se integra con Pull Request a `develop` |

- Cada Pull Request debe pasar el CI de GitHub Actions (lint con `ruff`, tests con `pytest` y build del frontend) antes de mergearse.
- Los commits siguen un formato simple: `feat:`, `fix:`, `chore:`, `docs:`.

## Enunciado del proyecto

### Contexto

R40 vende online mediante un `index.html` básico que se actualiza a mano: el cliente ve los productos y la venta se concreta por WhatsApp. Esto obliga a editar código para cada cambio de precio, no deja registro de las ventas y hace que cada compra dependa de una conversación manual.

La nueva página web reemplaza ese `index.html` por una tienda online completa: el cliente arma su carrito, elige el envío, paga y finaliza la compra en el sitio. El administrador gestiona precios, ofertas, pedidos y comprobantes desde un panel propio. WhatsApp se mantiene como canal de contacto y como alternativa de compra.

Este documento describe **qué** debe hacer el sistema (requerimientos funcionales y no funcionales) y **cómo** se va a ejecutar en 10 días.

### Stack tecnológico

| Capa | Tecnología | Motivo |
| --- | --- | --- |
| API | Python 3.12 + FastAPI | Simple de escribir, valida datos con Pydantic y genera documentación automática en `/docs` |
| Acceso a datos | SQLAlchemy 2.0 + Alembic | ORM maduro; Alembic versiona las migraciones de la base |
| Base de datos | PostgreSQL en Railway | Base relacional administrada, con backups |
| Frontend | React + Vite + React Router | SPA rápida; las páginas públicas se prerenderizan para SEO |
| Estilos | Tailwind CSS o Bootstrap (a definir) | Diseño responsivo sin escribir CSS desde cero |
| Pagos | Mercado Pago Checkout Pro (SDK oficial `mercadopago` de Python) | Medio de pago más usado en Argentina; cobro y redirección resueltos por Mercado Pago |
| Facturación | ARCA: WSAA + WSFEv1 (librería `pyafipws` o cliente SOAP `zeep`) | Python tiene las herramientas más probadas para ARCA |
| Imágenes | Cloudinary o Cloudflare R2 | Las imágenes no se guardan en la base ni en el servidor |
| Emails | Resend o Brevo | Confirmaciones de compra, envío y recuperación de contraseña |
| Despliegue | Railway, conectado al repositorio de GitHub | Deploy automático en cada push, sin Dockerfiles propios |

## Roles de usuario

El sistema distingue tres tipos de usuario:

| Rol | Descripción | Permisos principales |
| --- | --- | --- |
| **Visitante** | Cualquier persona que entra al sitio sin iniciar sesión | Navega el catálogo, arma el carrito, compra como invitado (sin cuenta), contacta por WhatsApp y lee "Quiénes somos" y FAQ |
| **Cliente** | Usuario registrado | Todo lo del visitante, más: guarda direcciones, ve el historial y el estado de sus pedidos, descarga sus comprobantes y conserva el carrito entre dispositivos |
| **Administrador** | Dueño o encargado de la tienda | Accede al panel admin: gestiona productos, precios, ofertas, stock, pedidos, envíos, comprobantes, FAQ y configuración general. Ve los reportes de ventas |

El login del administrador usa una ruta propia (`/admin/login`) y toda ruta del panel valida el rol en la API, no solo en el frontend.

## Parte 1 — API en Python

### Objetivo

Desarrollar una API REST con FastAPI que implemente toda la lógica de negocio de la tienda y persista la información en PostgreSQL. El frontend nunca habla directo con la base ni con Mercado Pago, ARCA o las empresas de envío: todo pasa por la API.

### Requerimientos funcionales

**Autenticación y usuarios**

- Registro e inicio de sesión con email y contraseña.
- Recuperación de contraseña mediante un link enviado por email, con vencimiento.
- El cliente puede editar sus datos y guardar una o más direcciones de envío.
- Un usuario desactivado no puede iniciar sesión.
- Solo un administrador puede crear otros administradores.

**Catálogo**

- ABM de categorías y productos, con al menos: nombre, slug (URL amigable), descripción, categoría, precio, imágenes, stock y estado (activo, oculto, próximamente).
- Los productos admiten **variantes** (por ejemplo talle y color), cada una con su SKU, precio y stock propios. Se diseña desde el inicio para la sección de ropa y equipaje.
- La categoría "Ropa y equipaje" se crea con estado **próximamente**: se muestra en el sitio pero sus productos no se pueden comprar.
- Listado con paginación, búsqueda por texto y filtros por categoría y rango de precio.

**Precios y ofertas**

- El administrador puede editar precios de forma individual o **masiva** (por ejemplo, aumentar un porcentaje a toda una categoría).
- Ofertas con: tipo (porcentaje o monto fijo), alcance (producto, variante o categoría), fecha de inicio y fecha de fin.
- Si hay más de una oferta vigente para un producto, se aplica la de mayor descuento (no se acumulan).
- Cada cambio de precio queda registrado con valor anterior, valor nuevo, usuario y fecha.

**Carrito**

- El visitante guarda el carrito en su navegador; el cliente logueado lo guarda en el servidor.
- Al iniciar sesión, el carrito del navegador se fusiona con el guardado.
- El precio final **siempre se calcula en la API** con el precio y las ofertas vigentes. Nunca se acepta un precio enviado por el frontend.
- Si un producto cambió de precio o se quedó sin stock, la API lo informa antes de pagar.

**Pedidos**

- Un pedido puede crearse con o sin cuenta (compra como invitado, pidiendo nombre, email, teléfono y DNI/CUIT).
- Cada ítem del pedido guarda una **copia** del nombre y precio al momento de la compra, para que un cambio posterior no altere ventas pasadas.
- Flujo de estados: **pendiente de pago → pagado → en preparación → enviado → entregado**, con **cancelado** posible desde pendiente o pagado.
- Al crear el pedido se reserva el stock durante 30 minutos. Si no se paga, la reserva se libera y el pedido pasa a cancelado.
- Cada cambio de estado notifica al cliente por email.

**Pagos**

- Integración con Mercado Pago Checkout Pro: la API crea la preferencia de pago y devuelve el link.
- Un endpoint de **webhook** recibe las notificaciones de Mercado Pago, verifica la firma y actualiza el pedido. El estado del pedido solo cambia por el webhook, nunca por la redirección del navegador.
- El webhook es **idempotente**: si llega la misma notificación dos veces, no se duplica el pago.
- Medio alternativo: transferencia bancaria. El pedido queda pendiente y el administrador confirma el pago a mano.

**Envíos**

- Cotización por código postal para Andreani y Correo Argentino.
- Si las credenciales de las APIs de envío no están disponibles al lanzar, se usa una **tabla de tarifas por zona** editable desde el admin.
- Opción de retiro en local (si aplica).
- El administrador carga el número de seguimiento y el cliente lo ve en "Mis pedidos".

**Comprobantes y facturación (ARCA)**

- Generación de un comprobante de venta en PDF para cada pedido pagado.
- Emisión de factura electrónica en ARCA: autenticación con WSAA y solicitud de CAE con WSFEv1.
- El tipo de comprobante depende de la condición fiscal de R40 (Factura C si es monotributista; A o B si es responsable inscripto) y del cliente.
- El PDF de la factura incluye CAE, fecha de vencimiento del CAE y el código QR exigido por ARCA.
- Exportación a Excel/CSV de las ventas de un período, como respaldo para carga manual.

**Contenido**

- ABM de preguntas frecuentes (pregunta, respuesta, orden).
- Configuración general editable: número de WhatsApp, datos bancarios, textos de "Quiénes somos", tarifas de envío.

**Reportes**

- Ventas por período (día, semana, mes) con total facturado, cantidad de pedidos y ticket promedio.
- Pedidos agrupados por estado.
- Productos más vendidos y productos con stock bajo.
- Los listados soportan filtros por fecha, estado y categoría, y paginación.

**SEO**

- Endpoint que genera el `sitemap.xml` con todas las páginas de productos y categorías activas.

### Requerimientos no funcionales

- **Seguridad**: contraseñas con hash `bcrypt` o `argon2`. Autenticación con JWT guardado en cookie `httpOnly`. Límite de intentos en el login. CORS restringido al dominio del frontend.
- **Validación de datos**: todos los datos se validan con Pydantic antes de guardarse; la API responde con errores claros (por ejemplo, cantidad mayor al stock disponible).
- **Trazabilidad**: cambios de precio, cambios de estado de pedidos y facturas emitidas quedan registrados en una tabla de auditoría.
- **Auditoría de registros**: toda tabla principal guarda fecha de creación, fecha de última modificación y el usuario que hizo cada una.
- **Migraciones**: todo cambio de la base se hace con una migración de Alembic versionada en el repositorio. Nunca se modifica la base a mano.
- **Documentación**: la documentación de la API (Swagger en `/docs`) debe estar disponible en staging.
- **Tests**: tests con `pytest` al menos para el cálculo de precios y ofertas, el flujo de estados del pedido y el webhook de pagos.

## Parte 2 — Frontend en React

### Objetivo

Desarrollar una aplicación web en React que consuma la API y ofrezca dos experiencias: la **tienda pública** para clientes y el **panel de administración** para gestionar el negocio.

### Requerimientos funcionales — Tienda

**Navegación y catálogo**

- Home con productos destacados, ofertas vigentes y acceso a categorías.
- Listado por categoría con filtros, orden (precio, novedades) y buscador.
- Ficha de producto con galería de imágenes, selector de variante, precio con oferta (precio anterior tachado) y botón de agregar al carrito.
- Sección "Ropa y equipaje" visible con el cartel **Próximamente**.

**Carrito y checkout**

- Carrito accesible desde cualquier página, con cantidad de ítems visible en el header.
- Checkout en pasos: datos del comprador → envío (cotización por código postal) → pago → confirmación.
- Opción de comprar sin crear cuenta.
- Páginas de resultado del pago: aprobado, pendiente y rechazado. La página de rechazo ofrece reintentar o **completar la compra por WhatsApp**.

**Cuenta del cliente**

- Registro, login, recuperación de contraseña.
- "Mi cuenta": datos personales y direcciones.
- "Mis pedidos": historial, estado, número de seguimiento y descarga de comprobantes.

**WhatsApp**

- Botón flotante visible en todo el sitio que abre un chat con R40 mediante un link `wa.me` con mensaje precargado.
- Botón **"Comprar por WhatsApp"** en el carrito, que arma el mensaje con el detalle de los productos, cantidades y total.

**Contenido institucional y legal**

- Página "Quiénes somos".
- Página de preguntas frecuentes (FAQ), con preguntas desplegables.
- Términos y condiciones, política de privacidad y datos del comercio.
- **Botón de arrepentimiento** visible en todo el sitio (obligatorio en Argentina para ventas online, Res. 424/2020).

**SEO**

- Las páginas públicas (home, categorías, productos, FAQ, quiénes somos) se **prerenderizan** a HTML para que Google las indexe correctamente.
- Cada página tiene título y meta descripción propios, URLs amigables (`/productos/nombre-del-producto`) y etiquetas Open Graph para que los links se vean bien al compartirlos por WhatsApp.
- Datos estructurados JSON-LD: `Product` en fichas, `FAQPage` en FAQ, `Organization` en el sitio.
- `robots.txt` y alta del sitio en Google Search Console con el `sitemap.xml`.
- Redirección 301 desde las direcciones del `index.html` viejo.

### Requerimientos funcionales — Panel de administración

- Login propio en `/admin/login`.
- **Dashboard** con ventas del día, semana y mes, ticket promedio, pedidos pendientes por estado, productos más vendidos y alertas de stock bajo.
- **Pedidos**: listado con filtros, detalle, cambio de estado, carga de número de seguimiento, confirmación manual de transferencias.
- **Productos**: ABM con variantes, stock e imágenes.
- **Precios y ofertas**: edición individual, aumento masivo por categoría y creación de ofertas con vigencia.
- **Comprobantes**: generar comprobante PDF, emitir factura en ARCA desde el pedido y exportar ventas a Excel.
- **Contenido**: edición de FAQ, "Quiénes somos", número de WhatsApp, datos bancarios y tarifas de envío.

### Requerimientos no funcionales

- **Diseño responsivo, mobile first**: la mayoría de las compras van a llegar desde el celular.
- **Rendimiento**: imágenes optimizadas y con carga diferida; la home debe cargar en menos de 3 segundos en una conexión 4G.
- **Manejo de sesión**: la sesión se mantiene de forma segura y finaliza al cerrar sesión o al expirar el token.
- **Feedback al usuario**: altas, ediciones, errores de validación y pagos se comunican con mensajes claros.
- **Estados de carga y error**: la aplicación informa cuando una operación está en curso y cuando falla la comunicación con la API.
- **Consistencia visual**: un mismo criterio de diseño (colores, tipografía, botones) en todo el sitio, alineado con la identidad de R40.

## Base de datos y despliegue

Todo corre en un solo proyecto de Railway con tres servicios (frontend, API y PostgreSQL) que se despliegan solos desde GitHub. La API es el único componente que habla con la base y con los servicios externos.

&#91;embedded content: arquitectura · frontend, API, base y servicios externos\]

El navegador solo habla con el frontend y con WhatsApp; los pagos, envíos, facturas, imágenes y emails se resuelven siempre desde la API.

### Modelo de datos

| Tabla | Contenido principal |
| --- | --- |
| `users` | Email, hash de contraseña, nombre, teléfono, DNI/CUIT, rol, activo |
| `addresses` | Direcciones de envío de cada cliente |
| `categories` | Nombre, slug, estado (activa, próximamente) |
| `products` | Nombre, slug, descripción, categoría, estado |
| `product_variants` | SKU, talle, color, precio, stock |
| `product_images` | URL de la imagen, orden, variante asociada |
| `promotions` | Tipo, valor, alcance, fecha de inicio y fin |
| `price_history` | Precio anterior, precio nuevo, usuario, fecha |
| `carts` / `cart_items` | Carrito guardado del cliente logueado |
| `orders` | Comprador, estado, totales, método de envío, dirección |
| `order_items` | Copia de nombre, variante, cantidad y precio al comprar |
| `payments` | Medio, ID de Mercado Pago, estado, monto, notificaciones recibidas |
| `shipments` | Empresa, costo, número de seguimiento, estado |
| `invoices` | Tipo de comprobante, punto de venta, número, CAE, vencimiento, PDF |
| `shipping_rates` | Tarifas por zona (plan B de envíos) |
| `faq_items` | Pregunta, respuesta, orden |
| `settings` | WhatsApp, datos bancarios, textos institucionales |
| `audit_log` | Acción, entidad, usuario, fecha, detalle |

Todas las tablas principales incluyen `created_at`, `updated_at`, `created_by` y `updated_by`.

### Despliegue en Railway

- **PostgreSQL**: servicio administrado de Railway con backups activados. La API lo lee con la variable `DATABASE_URL` (referenciada como `${{Postgres.DATABASE_URL}}`). SQLAlchemy necesita el prefijo `postgresql+psycopg://`, así que la configuración lo convierte al iniciar.
- **API**: Railway detecta Python por el `requirements.txt` de `api/` (Root Directory = `api`). Comando de inicio: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`. Comando previo al deploy: `alembic upgrade head`, para que las migraciones corran solas en cada despliegue.
- **Frontend**: Root Directory = `frontend`, build con `npm run build` y servido como sitio estático. La URL de la API se pasa con la variable `VITE_API_URL`.
- **Variables de entorno** (solo en Railway, nunca en el repo): `JWT_SECRET`, `MP_ACCESS_TOKEN`, `MP_WEBHOOK_SECRET`, credenciales de envíos, certificado y clave de ARCA, credenciales de imágenes y emails.
- **Dominio**: el dominio actual apunta al servicio frontend; la API usa un subdominio (por ejemplo `api.<dominio>`). Railway gestiona el certificado HTTPS.

### Desarrollo local

1. `docker compose up -d` levanta PostgreSQL con volumen persistente.
2. En `api/`: crear el entorno virtual, `pip install -r requirements.txt`, `alembic upgrade head` y `uvicorn app.main:app --reload`.
3. En `frontend/`: `npm install` y `npm run dev`.
4. Mercado Pago se prueba con credenciales y usuarios de prueba; ARCA con el entorno de homologación.

## Plan de ejecución (10 días)

Cada día termina con algo desplegado en staging. El día 10 el sitio pasa a producción y reemplaza al `index.html`.

### Día 0 — Trámites que se inician antes de programar

- [ ] Mercado Pago: cuenta vendedor, crear la aplicación en el panel de desarrolladores, obtener credenciales de prueba y crear usuarios de prueba.
- [ ] Andreani: solicitar alta como cliente corporativo y acceso a la API.
- [ ] Correo Argentino: alta en MiCorreo y solicitud de credenciales de la API.
- [ ] ARCA: confirmar la condición fiscal de R40, generar el certificado para homologación, asociarlo al servicio `wsfe` y crear un punto de venta para Web Services.
- [ ] Dominio: confirmar quién lo administra y conseguir acceso al DNS.
- [ ] Contenido: fotos, descripciones y precios actuales (extraer del `index.html`), texto de "Quiénes somos" y preguntas frecuentes que llegan por WhatsApp.

### Día 1 — Infraestructura y esqueleto

- [ ] Crear el repo `r40-web` con `api/`, `frontend/`, `docker-compose.yml` y `README.md`.
- [ ] API: proyecto FastAPI con estructura `routers/`, `models/`, `schemas/`, `services/`; configuración con `pydantic-settings`; endpoint `/health`.
- [ ] Frontend: proyecto Vite + React + React Router con layout base (header, footer).
- [ ] Railway: proyecto con PostgreSQL, servicio `api` y servicio `frontend`, entornos staging y producción.
- [ ] Primera migración de Alembic con el modelo de datos completo y script de seed con los productos del `index.html`.
- [ ] GitHub Actions: `ruff`, `pytest` y build del frontend en cada Pull Request.

* **Entregable:** API y frontend desplegados en staging y conectados a la base.

### Día 2 — Catálogo

- [ ] Endpoints de categorías, productos, variantes e imágenes con paginación, búsqueda y filtros.
- [ ] Subida de imágenes a Cloudinary o R2.
- [ ] Frontend: home, listado por categoría, ficha de producto, buscador.
- [ ] Categoría "Ropa y equipaje" con estado próximamente.

* **Entregable:** catálogo navegable en staging con los productos reales.

### Día 3 — Usuarios y autenticación

- [ ] Registro, login, logout y JWT en cookie `httpOnly`.
- [ ] Recuperación de contraseña por email (Resend o Brevo).
- [ ] Rol administrador, dependencia de FastAPI que protege rutas admin y límite de intentos de login.
- [ ] Frontend: login, registro, "Mi cuenta", direcciones y `/admin/login`.

* **Entregable:** clientes y administrador inician sesión; las rutas admin están protegidas.

### Día 4 — Carrito y pedidos

- [ ] Carrito en navegador para invitados y en servidor para clientes, con fusión al loguearse.
- [ ] Servicio de precios: precio vigente + mejor oferta activa, calculado siempre en la API.
- [ ] Creación de pedidos con copia de precios, reserva de stock por 30 minutos y flujo de estados.
- [ ] Tarea programada que cancela pedidos impagos y libera stock.
- [ ] Frontend: carrito, checkout (datos del comprador) y compra como invitado.

* **Entregable:** se arma un carrito y se genera un pedido pendiente de pago.

### Día 5 — Pagos

- [ ] Crear la preferencia de Mercado Pago con el SDK de Python y las URLs de retorno.
- [ ] Webhook con verificación de firma e idempotencia; actualiza el pedido a pagado.
- [ ] Opción de transferencia bancaria con confirmación manual.
- [ ] Email de confirmación de compra.
- [ ] Frontend: páginas de pago aprobado, pendiente y rechazado.

* **Entregable:** compra completa con usuarios de prueba de Mercado Pago.

### Día 6 — Envíos

- [ ] Cotización por código postal en el checkout.
- [ ] Si hay credenciales: integrar cotización y alta de envío con Andreani y MiCorreo.
- [ ] Si no hay credenciales: tabla de tarifas por zona editable y carga manual del seguimiento.
- [ ] Opción de retiro en local.

* **Entregable:** el checkout calcula el envío y el pedido guarda método y seguimiento.

### Día 7 — Panel de administración

- [ ] Dashboard de ventas: totales por período, ticket promedio, pedidos por estado, más vendidos, stock bajo.
- [ ] Gestión de pedidos: detalle, cambio de estado, seguimiento y email al cliente.
- [ ] ABM de productos, variantes, stock e imágenes.
- [ ] Aumento masivo de precios por categoría y ofertas con vigencia.
- [ ] Tabla de auditoría de cambios.

* **Entregable:** el administrador opera la tienda sin tocar código.

### Día 8 — Comprobantes y ARCA

- [ ] Comprobante de venta en PDF (con `reportlab` o `weasyprint`).
- [ ] WSAA: firma del ticket de acceso con el certificado y cacheo del token (dura 12 horas).
- [ ] WSFEv1: emisión de factura en homologación y obtención del CAE.
- [ ] PDF de factura con CAE, vencimiento y código QR.
- [ ] Exportación de ventas a Excel/CSV como plan B.

* **Entregable:** facturas emitidas en homologación desde el panel.

### Día 9 — Contenido, WhatsApp, legales y SEO

- [ ] Páginas "Quiénes somos" y FAQ, editables desde el admin.
- [ ] Botón flotante de WhatsApp y "Comprar por WhatsApp" desde el carrito.
- [ ] Botón de arrepentimiento, términos y condiciones, política de privacidad.
- [ ] Prerender de páginas públicas, títulos y meta descripciones, JSON-LD, Open Graph, `sitemap.xml` y `robots.txt`.
- [ ] Redirecciones 301 desde el `index.html` viejo.

* **Entregable:** sitio completo en staging.

### Día 10 — Pruebas y lanzamiento

- [ ] Pruebas de punta a punta en celular y computadora: Mercado Pago, transferencia, compra como invitado, cancelación, arrepentimiento.
- [ ] Revisión de seguridad: rutas admin, validaciones, secretos solo en Railway, HTTPS.
- [ ] Backups de PostgreSQL activos y una prueba de restauración.
- [ ] Credenciales de producción de Mercado Pago, apuntar el DNS al frontend en Railway.
- [ ] Compra real de bajo monto para validar todo el circuito.
- [ ] Alta en Google Search Console y monitoreo de disponibilidad (UptimeRobot).

* **Entregable:** sitio en producción reemplazando al `index.html`.

## Riesgos y fase 2

El mayor riesgo del plazo no es el código sino las integraciones que dependen de terceros. Cada una tiene un plan B que permite lanzar igual.

| Riesgo | Impacto | Plan B |
| --- | --- | --- |
| Andreani o Correo Argentino no entregan credenciales a tiempo | No hay cotización automática de envío | Tabla de tarifas por zona y carga manual del seguimiento |
| Certificado o homologación de ARCA demoran | No se emiten facturas desde el sitio | Comprobante PDF interno y exportación a Excel para facturar a mano |
| Mercado Pago no habilita credenciales de producción | No se cobra online | Transferencia bancaria y "Comprar por WhatsApp" |
| El prerender no queda listo | Google indexa mal los productos | Lanzar igual y completar SEO en la primera semana |
| Faltan fotos o textos | Catálogo incompleto | Lanzar con los productos que ya estaban en el `index.html` |

### Imprescindible para lanzar

Catálogo, carrito, Mercado Pago, cotización de envío (aunque sea por tabla), panel de pedidos y precios, WhatsApp, botón de arrepentimiento y legales.

### Fase 2 (después del lanzamiento)

- Integración automática completa con Andreani y Correo Argentino (alta de envío y etiquetas).
- ARCA en producción.
- Lanzamiento de la sección de ropa y equipaje con sus variantes.
- Cupones de descuento y medios de pago adicionales.
- Analítica (Google Analytics 4) y Google Merchant Center para mostrar productos en Google Shopping.

### Decisiones pendientes

- [ ] Condición fiscal de R40 (define el tipo de factura).
- [ ] Integrantes del equipo y quién toma cada parte.
- [ ] Tailwind o Bootstrap para los estilos.
- [ ] Cloudinary o Cloudflare R2 para las imágenes.
- [ ] Si se ofrece retiro en local y desde qué dirección.
