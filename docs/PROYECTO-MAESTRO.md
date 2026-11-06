# Custom Line — Documento maestro del proyecto

> Documento de arranque para reconstruir Custom Line desde cero, sin ayuda de IA.
> La especificación técnica detallada vive en `docs/SPEC.md` (fuente de verdad del modelo de datos,
> endpoints y reglas). Este documento es el "qué" y el "por qué"; la SPEC es el "cómo exacto".

---

## 1. De qué trata el proyecto

**Custom Line** es una plataforma web (tienda online + panel de administración/producción) para un negocio
de **personalización de prendas y artículos**: camisetas, buzos, chompas, mugs, termos, carcasas de celular,
mouse pads, llaveros, etc.

El negocio imprime diseños sobre esos productos con dos técnicas:

- **Sublimación:** solo sobre prendas blancas con ≥ 70 % poliéster y artículos con recubrimiento de poliéster
  (mugs, carcasas, termos…). Cada producto tiene sus parámetros (temperatura, tiempo, presión).
- **DTF (Direct to Film):** solo prendas, funciona en casi cualquier tela apta al calor. Proceso estándar,
  pero se guardan parámetros por si cambian.

El **corazón del sistema es la personalización**: el cliente diseña sobre el producto en un editor 2D, lo
previsualiza en 3D, y el sistema genera un **PNG a tamaño real (300 DPI, fondo transparente)** por zona, listo
para que producción lo imprima. También hay productos terminados que se compran directo, sin personalizar.

### Las dos caras de la plataforma

1. **Cliente:** catálogo → elegir producto, color y talla → personalizador 2D/3D → carrito → checkout →
   pago → seguimiento del pedido e historial.
2. **Producción / administración:** dashboard de pedidos, archivos listos para imprimir, inventario de
   productos e insumos, proveedores, compras, precios, cotizaciones, facturación interna, gastos, reportes de
   ganancias, incidentes/reprocesos y envíos.

### Principio rector de finanzas

La prueba de concepto en Excel calculaba precios pero "no se veía la plata". El sistema debe responder siempre:
**¿cuánto entró, cuánto salió y cuánto quedó — por pedido, por producto y por mes?**
Por eso se guarda el **costo estimado** al cotizar y el **costo real** al producir (insumos reales, reprocesos,
imprevistos) y se registran todos los gastos.

---

## 2. Alcance

### Incluido (MVP + fases de este kit)

- Cuentas con roles (`ADMIN`, `EMPLEADO`, `CLIENTE`; un usuario puede tener varios).
- Catálogo con productos, variantes (color/talla/material) y zonas imprimibles con medidas en cm por talla.
- Personalizador 2D (fuente de verdad del diseño) y visor 3D (misma información, solo vista previa).
- Carga de imágenes y textos por zona; comentario libre del cliente por ítem.
- Validación de resolución (DPI efectivo) con aviso al cliente y alerta al productor.
- Generación automática de **PNG a tamaño real, 300 DPI, fondo transparente**, por zona.
- Mejora de resolución opcional con Real-ESRGAN (worker con GPU) cuando el DPI es bajo.
- Carrito con productos personalizados y terminados; pedido como unidad (un estado, un envío).
- Pagos **manuales** (transferencia, Nequi, efectivo) confirmados por un empleado; **Wompi preparado** detrás
  de una interfaz `PaymentGateway`, apagado por configuración.
- Factura **interna** (consecutivo propio + PDF). Interfaz `InvoiceProvider` para DIAN a futuro.
- Inventario de productos (por variante) e insumos, movimientos de stock, alertas de stock bajo.
- Proveedores, compras y gastos (luz, hosting, arriendo, etc.).
- Motor de precios con **fórmula configurable y versionada** desde el panel.
- Producción: asignación de pedido a un empleado, parámetros de técnica visibles, trazabilidad.
- Incidentes/reprocesos: descuentan stock y registran la pérdida en dinero.
- Envíos: transportadoras, guía, costo, estado.
- Notificaciones **solo dentro de la plataforma** (campana en panel y cuenta del cliente).
- Reportes: ingresos vs. gastos reales por mes, rentabilidad por producto y por pedido.
- Despliegue en un PC propio con Docker Compose + Cloudflare Tunnel + backups fuera del PC.

### Fuera de alcance por ahora (dejar extensible, no construir)

- Factura electrónica DIAN (solo interfaz).
- Wompi en producción (implementado pero apagado).
- Corte láser, toldos y otros negocios del Excel original.
- App nativa / Flutter (la web responsive cubre celulares).
- Notificaciones por correo o WhatsApp (dejar el `NotificationChannel` extensible).
- Producción por etapas o tercerizada (hoy: un pedido → un responsable).

---

## 3. Stack tecnológico

| Capa | Tecnología |
|---|---|
| Backend | Java 21, Spring Boot 3.x, Spring Web, Spring Security (JWT access + refresh), Spring Data JPA, Bean Validation, Flyway, springdoc-openapi, MapStruct |
| Base de datos | PostgreSQL 16 |
| Tests backend | JUnit 5, Testcontainers (Postgres), MockMvc / RestAssured |
| Frontend | React 18 + TypeScript + Vite, React Router, TanStack Query, Zustand, Tailwind CSS, react-konva (editor 2D), @react-three/fiber + drei (3D), MSW (mocks), Vitest + Testing Library, Playwright (E2E) |
| Imágenes | Render de PNG en el backend (Java2D/ImageIO a 300 DPI con metadatos pHYs). Upscaling: worker Python con `realesrgan-ncnn-vulkan` |
| Archivos | Disco local (`/data/files`) detrás de una interfaz `FileStorage` (para pasar a S3/R2 después) |
| Infra | Docker Compose, Nginx (sirve el frontend y hace proxy a `/api`), Cloudflare Tunnel, `pg_dump` + `rclone` para backups |

**Convenciones transversales:**
- Moneda: **COP en enteros** (`long`, pesos sin decimales). Nunca `double` para dinero.
- Zona horaria `America/Bogota`. Idioma de la UI: **español**. Código/comentarios: **inglés**.
- Medidas físicas en **centímetros** (`numeric(6,2)`); temperaturas en **°F** (así las usa el taller).

---

## 4. Arquitectura (monorepo)

```
custom-line/
├── docs/            # SPEC.md, openapi.yaml, este documento
├── backend/         # Spring Boot, paquete raíz co.customline, paquete por módulo
│   └── .../common, auth, catalog, inventory, pricing, design,
│          orders, payments, billing, production, shipping, finance, notifications
├── frontend/        # React + TS + Vite
│   └── src/app, shared, features/{auth,store,account,editor2d,viewer3d,admin}, mocks
├── workers/upscaler/# servicio Python (FastAPI) de mejora de resolución
└── infra/           # docker-compose, nginx, cloudflared, backups
```

**Reglas de arquitectura backend:**
- Paquete por módulo. Dentro de cada uno: `api` (controllers + DTOs), `domain` (entidades, reglas),
  `repo`, `service`.
- Los módulos se comunican por **servicios públicos o eventos de Spring**, nunca tocando el repositorio de
  otro módulo.
- DTOs explícitos (records) + MapStruct. **Nunca** exponer entidades JPA en la API.
- Errores con `ProblemDetail` (RFC 7807) y códigos propios (`CATALOG_VARIANT_NOT_FOUND`).
- Transacciones en la capa de servicio. El movimiento de stock y la actualización de `stock_qty` van en la
  misma transacción.

**Reglas de arquitectura frontend:**
- TypeScript estricto. TanStack Query para datos del servidor, Zustand para el estado del editor.
- Tipos generados del OpenAPI en `src/shared/api/generated/` (no editar a mano).
- Mobile-first: probar en 375 px y en escritorio.
- El frontend trabaja contra **mocks de MSW** generados del contrato hasta que el backend esté listo.

---

## 5. Modelo de datos (resumen; detalle en SPEC §5)

- **Usuarios/roles:** `users`, `roles`, `user_roles`, `addresses`, `refresh_tokens`.
- **Catálogo:** `categories`, `techniques`, `products`, `product_techniques`, `product_variants`,
  `print_zones`, `print_zone_sizes`, `technique_parameters`.
- **Inventario:** `suppliers`, `supplies`, `supply_consumption_rules`, `stock_movements`, `purchases`,
  `purchase_items`.
- **Diseño:** `design_assets`, `designs` (JSON `data` versión 1), `production_files`, `files`.
- **Pedidos:** `carts`, `cart_items`, `orders`, `order_items`, `order_status_history`.
- **Pagos/facturación:** `payments`, `invoices`, `quotes`.
- **Producción:** `incidents`, `incident_items` (asignación vía `orders.assigned_to`).
- **Envíos:** `carriers`, `shipments`, `shipping_rates`.
- **Finanzas:** `expense_categories`, `expenses` (reportes calculados, no tablas).
- **Precios:** `pricing_formulas`, `pricing_components`, `product_pricing_overrides`.
- **Notificaciones/auditoría:** `notifications`, `audit_log`.

**Estados del pedido:** `PENDIENTE_PAGO → PAGADO → EN_PRODUCCION → LISTO → ENVIADO → ENTREGADO`,
y `CANCELADO` desde `PENDIENTE_PAGO`/`PAGADO` (devuelve stock). Validados por máquina de estados en el backend.

**JSON de diseño (versión 1):** zonas con capas (`image`/`text`), coordenadas en cm con origen en la esquina
superior izquierda. El editor 2D y el visor 3D leen y escriben **este mismo JSON**; el 3D nunca es fuente de
verdad. DPI efectivo de una imagen = `min(width_px / (widthCm/2.54), height_px / (heightCm/2.54))`
(≥300 ok, 150–299 aviso amarillo, <150 aviso rojo).

---

## 6. Flujos principales

**Cliente personaliza y compra:** catálogo → producto (color/talla) → personalizador (sube/escribe por zona,
mueve/escala/rota, alterna 2D/3D, ve DPI) → comentario → carrito (diseño `SUBMITTED`) → checkout (dirección o
recogida + método de pago) → pedido `PENDIENTE_PAGO` con **precio congelado** → sube comprobante → empleado
confirma → `PAGADO` → seguimiento en "Mis pedidos".

**Producción:** notificación "pedido nuevo" → empleado lo toma (se asigna) → ve cada ítem con sus PNG por zona,
técnica y parámetros → imprime → si falla, registra incidente (descuenta stock + suma pérdida) → `LISTO`
(descuenta insumos) → crea envío con guía → `ENVIADO` → `ENTREGADO`. Al pasar a `PAGADO` se emite factura interna.

**Admin:** gestiona catálogo, variantes, zonas/medidas, parámetros de técnica, insumos, proveedores, compras,
empleados, transportadoras, gastos, fórmulas de precio y reportes.

---

## 7. Motor de precios (clave del negocio)

Fórmula **configurable y versionada** desde el panel. Cada `order_item` guarda su `price_breakdown`
(componente → valor) y la versión de fórmula usada; cambiar la fórmula **nunca** altera pedidos viejos.

Componentes (`kind`): `VARIANT_PURCHASE_COST`, `LABOR` (valor hora × minutos), `FIXED_PER_UNIT`,
`PRINT_DTF_CM2`, `PRINT_SUBLIMATION_CM2`, `SUPPLIES_ESTIMATED`, `OVERHEAD_PER_ORDER`, `PERCENT_OF_COST`,
`PER_EXTRA_ELEMENT`, `GATEWAY_FEE` (con gross-up para Wompi).

**Fórmula v1 (semilla del Excel):**
```
costo  = compra + valor_hora × (h_diseño + h_fabricación) + envío_taller
precio = ROUNDDOWN_1000( costo × (1.10 + imprevistos + administración + utilidad) )
```
Costo DTF de referencia: metro 58×100 cm = 5.800 cm² a $23.000 → **$3,97/cm²**;
área cobrada = `(ancho + 2·margen) × (alto + 2·margen)`, margen 1 cm.

---

## 8. Seguridad

- Contraseñas con BCrypt. JWT de acceso corto (15 min) + refresh rotativo.
- Autorización por rol en cada endpoint. Un cliente solo ve sus pedidos, diseños y archivos.
- URLs de archivos firmadas o vía endpoint que valida permisos.
- Validar tipo y tamaño de subidas (PNG, JPG, WEBP, SVG sanitizado, PDF; máx. 25 MB); recalcular dimensiones
  en el servidor.
- Rate limit en login y subida de archivos. Secretos solo por variables de entorno.

---

## 9. Plan de construcción (orden recomendado para hacerlo solo)

Como lo vas a hacer **una sola persona y en serie** (no en paralelo con varios agentes), el orden sugerido es:

1. **Fundación:** repositorio, estructura del monorepo, Docker base de Postgres, backend que arranca, frontend
   que arranca, CI mínimo.
2. **Esquema + auth:** Flyway `V1__schema.sql` + `V2__seed.sql`, registro/login con JWT y roles.
3. **Catálogo + inventario:** productos, variantes, zonas, técnicas, stock, movimientos.
4. **Precios + finanzas:** fórmula versionada, simulación de precio, gastos y reportes.
5. **Diseño:** assets, modelo de `designs`, validación de DPI, render del PNG de producción.
6. **Editor 2D** y **visor 3D** en el frontend (leen/escriben el mismo JSON).
7. **Tienda cliente:** catálogo, ficha de producto, carrito, checkout.
8. **Pedidos + pagos + producción + envíos:** máquina de estados, pagos manuales, incidentes, guías.
9. **Panel admin:** todas las pantallas de administración/producción.
10. **Notificaciones + auditoría.**
11. **Upscaler** (worker Python) e integración asíncrona con el render.
12. **Despliegue:** docker-compose completo, Nginx, Cloudflare Tunnel, backups.
13. **QA / E2E** del flujo cliente→producción y pulido responsive.

La lista de micro-tareas y subtareas está en `docs/tareas-custom-line.json`, lista para importar.

---

## 10. Definición de "terminado" para cualquier tarea

- Compila, tests pasan, lint sin errores.
- Endpoint documentado en `openapi.yaml` (si es backend) o pantalla responsive verificada (si es frontend).
- Sin TODOs silenciosos.