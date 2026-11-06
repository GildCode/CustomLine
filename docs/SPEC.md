# Custom Line — Especificación del sistema

> Fuente de verdad del proyecto. Si un prompt o una decisión contradice este archivo, gana este archivo
> salvo que un archivo de `memory/decisions/` registre el cambio con fecha y motivo.

## 1. Qué es

Custom Line es un negocio de personalización de prendas y artículos (camisetas, buzos, chompas, mugs,
termos, carcasas de celular, mouse pads, llaveros, etc.) que trabaja con dos técnicas:

- **Sublimación:** solo en prendas blancas con ≥ 70 % poliéster y en artículos con recubrimiento de poliéster
  (mugs, carcasas, mouse pads, termos…). Se imprime con tinta de sublimación, se fija con cinta y se aplica
  calor con presión. Cada producto tiene parámetros propios (temperatura, tiempo, presión).
- **DTF (Direct to Film):** solo prendas, sirve en casi cualquier tela apta al calor. Proceso estándar
  (misma temperatura y tiempo), pero se guardan parámetros por si cambia.

La plataforma es una **tienda online responsive** con dos caras:

1. **Cliente:** catálogo, personalizador 2D + 3D, carrito, pedidos, cuenta, historial y estado de pedidos.
2. **Producción / administración:** dashboard de pedidos, archivos listos para imprimir, inventario de
   productos e insumos, proveedores, precios, cotizaciones, facturación interna, gastos, reportes de
   ganancias, incidentes y envíos.

El núcleo del negocio es la **personalización**. También hay una sección de **productos terminados** que se
compran directo.

### Principio rector de finanzas
La prueba de concepto en Excel calculaba precios pero "no se vio la plata". El sistema debe responder siempre:
**¿cuánto entró, cuánto salió y cuánto quedó, por pedido, por producto y por mes?** Por eso se guarda el
costo **estimado** al cotizar y el costo **real** al producir (insumos reales, reprocesos, imprevistos), y
se registran todos los gastos.

## 2. Alcance

### Incluido (MVP + fases de este kit)
- Cuentas de usuario con roles (`ADMIN`, `EMPLEADO`, `CLIENTE`; un usuario puede tener varios).
- Catálogo con productos, variantes (color/talla/material), zonas imprimibles con medidas en cm por talla.
- Personalizador 2D (fuente de verdad del diseño) y visor 3D (vista previa del mismo diseño).
- Carga de imágenes y textos por zona; comentario libre del cliente por ítem.
- Validación de resolución (DPI efectivo) con aviso al cliente y alerta al productor.
- Generación automática de **PNG a tamaño real, 300 DPI, fondo transparente**, por zona, listo para imprimir.
- Mejora de resolución opcional con Real-ESRGAN (worker con GPU) cuando el DPI efectivo es bajo.
- Carrito con productos personalizados y terminados; pedido como unidad (un estado, un envío).
- Pagos **manuales** (transferencia, Nequi, efectivo) confirmados por un empleado; **Wompi preparado** detrás
  de una interfaz `PaymentGateway`, desactivado por configuración.
- Factura **interna** (consecutivo propio + PDF). Interfaz `InvoiceProvider` para conectar DIAN a futuro.
- Inventario de productos (por variante) e insumos, movimientos de stock, alertas de stock bajo.
- Proveedores, compras, gastos (luz, hosting, arriendo, etc.).
- Motor de precios con **fórmula configurable y versionada** desde el panel.
- Producción: asignación de pedido a un empleado, parámetros de técnica visibles, registro de quién hizo qué.
- Incidentes/reprocesos: descuentan stock y registran pérdida en dinero.
- Envíos: empresas transportadoras, guía, costo, estado.
- Notificaciones **solo dentro de la plataforma** (campana en el panel y en la cuenta del cliente).
- Reportes: ingresos vs. gastos reales por mes, rentabilidad por producto y por pedido.
- Despliegue en un PC propio con Docker Compose + Cloudflare Tunnel + backups automáticos fuera del PC.

### Fuera de alcance por ahora (dejar extensible, no construir)
- Factura electrónica DIAN (solo interfaz).
- Wompi en producción (implementación lista pero apagada).
- Corte láser, toldos y otros negocios de las hojas del Excel.
- App nativa / Flutter (la web responsive cubre celulares).
- Notificaciones por correo o WhatsApp (dejar el `NotificationChannel` extensible).
- Producción por etapas o tercerizada (hoy: un pedido → un responsable).

## 3. Stack

| Capa | Tecnología |
|---|---|
| Backend | Java 21, Spring Boot 3.x, Spring Web, Spring Security (JWT access + refresh), Spring Data JPA, Bean Validation, Flyway, springdoc-openapi, MapStruct, Lombok opcional |
| Base de datos | PostgreSQL 16 |
| Tests backend | JUnit 5, Testcontainers (Postgres), MockMvc / RestAssured |
| Frontend | React 18 + TypeScript + Vite, React Router, TanStack Query, Zustand, Tailwind CSS, react-konva (editor 2D), @react-three/fiber + @react-three/drei (3D), MSW (mocks), Vitest + Testing Library, Playwright (E2E) |
| Imágenes | Render de PNG de producción en el backend (Java2D/ImageIO a 300 DPI con metadatos pHYs). Upscaling: worker Python con `realesrgan-ncnn-vulkan` |
| Archivos | Disco local montado (`/data/files`), detrás de una interfaz `FileStorage` (para poder pasar a S3/R2 después) |
| Infra | Docker Compose, Nginx (sirve el frontend y hace proxy a `/api`), Cloudflare Tunnel, `pg_dump` + `rclone` para backups |

Moneda: **COP en enteros** (`long`, pesos sin decimales). Zona horaria: `America/Bogota`. Idioma de la UI: español.
Medidas físicas en **centímetros** (`numeric(6,2)`), temperaturas en **°F** (así las usa el taller).

## 4. Estructura del repositorio (monorepo)

```
custom-line/
├── CLAUDE.md                  # reglas para todos los agentes (lo carga Claude Code)
├── docs/
│   ├── SPEC.md                # este archivo
│   └── openapi.yaml           # contrato de la API (lo crea la Fundación; dueño: ver §12)
├── memory/                    # memoria persistente de los agentes (ver CLAUDE.md)
│   ├── PROGRESS.md            # generado por scripts/progress.sh
│   ├── decisions/             # un archivo por decisión
│   ├── requests/              # peticiones entre agentes, un archivo cada una
│   └── agents/<agente>.md     # memoria de cada agente
├── scripts/progress.sh
├── backend/                   # Spring Boot
│   └── src/main/java/co/customline/
│       ├── common/            # errores, paginación, auditoría, money, config
│       ├── auth/              # usuarios, roles, JWT
│       ├── catalog/           # categorías, productos, variantes, zonas, técnicas, parámetros
│       ├── inventory/         # stock, insumos, proveedores, movimientos, alertas
│       ├── pricing/           # fórmulas versionadas, cálculo de precio
│       ├── design/            # assets, diseños, render PNG, validación DPI
│       ├── orders/            # carrito, pedidos, historial de estados
│       ├── payments/          # pagos manuales, PaymentGateway (Wompi)
│       ├── billing/           # facturas internas, cotizaciones
│       ├── production/        # asignación, incidentes, costo real
│       ├── shipping/          # transportadoras, envíos
│       ├── finance/           # gastos, reportes
│       └── notifications/     # notificaciones in-app
├── frontend/
│   └── src/
│       ├── app/               # router, providers, layout
│       ├── shared/            # api client, ui kit, hooks, tipos generados
│       ├── features/
│       │   ├── auth/
│       │   ├── store/         # catálogo, producto, carrito, checkout (cliente)
│       │   ├── account/       # mis pedidos, perfil, direcciones
│       │   ├── editor2d/      # personalizador 2D
│       │   ├── viewer3d/      # visor 3D
│       │   └── admin/         # producción, inventario, precios, finanzas, etc.
│       └── mocks/             # MSW handlers
├── workers/upscaler/          # servicio Python de mejora de resolución
└── infra/                     # docker-compose, nginx, cloudflared, backups
```

## 5. Modelo de datos

Convenciones: tablas en `snake_case` plural, `id bigserial`, `created_at`/`updated_at timestamptz`,
dinero en `bigint` (COP), medidas en `numeric(6,2)` (cm). Borrado lógico con `active boolean` en catálogos.

### 5.1 Usuarios y roles
- `users` (id, email único, password_hash, full_name, phone, active, created_at)
- `roles` (id, code: `ADMIN|EMPLEADO|CLIENTE`)
- `user_roles` (user_id, role_id) — N:M
- `addresses` (id, user_id, label, city, department, address_line, phone, notes, is_default)
- `refresh_tokens` (id, user_id, token_hash, expires_at, revoked)

Los empleados se crean desde el panel admin. El registro público siempre crea `CLIENTE`.

### 5.2 Catálogo
- `categories` (id, name, slug, parent_id nullable, sort_order)
- `techniques` (id, code: `SUBLIMATION|DTF`, name, config jsonb)
  - DTF `config` ejemplo: `{"sheetWidthCm":58,"sheetLengthCm":100,"sheetCost":23000,"marginCm":1.0}`
- `products` (id, category_id, name, slug, description, kind: `CUSTOMIZABLE|FINISHED`, sublimable boolean,
  model3d_url nullable, preview_images jsonb, active)
- `product_techniques` (product_id, technique_id) — qué técnicas admite
- `product_variants` (id, product_id, sku único, color_name, color_hex, size, material, polyester_pct,
  purchase_cost bigint, sale_price_override bigint nullable, supplier_id, stock_qty int, min_stock int, active)
  - Cada combinación color/talla/material es una variante (mug blanco ≠ mug rojo; S ≠ XL).
- `print_zones` (id, product_id, code: `FRONT|BACK|SLEEVE_L|SLEEVE_R|WRAP|...`, name, uv_rect jsonb para 3D,
  overlay_image_url para 2D)
- `print_zone_sizes` (id, print_zone_id, size, max_width_cm, max_height_cm) — medidas reales por talla
  (editables; arrancar con estándar).
- `technique_parameters` (id, technique_id, product_id nullable = default de la técnica, temperature_f,
  time_seconds, pressure, notes)

### 5.3 Inventario
- `suppliers` (id, name, contact_name, phone, email, notes, active)
- `supplies` (id, name, unit: `ML|SHEET|M|UNIT|CM2|G`, unit_cost bigint, stock_qty numeric, min_stock numeric,
  supplier_id, active) — tinta, papel de sublimación, film DTF, cinta, empaques…
- `supply_consumption_rules` (id, product_id nullable, technique_id nullable, supply_id, qty_per_unit numeric,
  qty_per_cm2 numeric) — consumo **estimado**
- `stock_movements` (id, item_type: `VARIANT|SUPPLY`, item_id, qty numeric (+/-), reason:
  `PURCHASE|SALE|ADJUSTMENT|INCIDENT|REPROCESS|RETURN`, unit_cost bigint, order_id nullable,
  incident_id nullable, user_id, note, created_at)
  - El stock actual vive en `stock_qty` y se actualiza en la misma transacción que el movimiento.
- `purchases` (id, supplier_id, date, total bigint, invoice_ref, user_id) + `purchase_items`
  (purchase_id, item_type, item_id, qty, unit_cost)

### 5.4 Diseño
- `design_assets` (id, owner_user_id, file_id, original_name, mime, width_px, height_px, sha256,
  upscaled_file_id nullable, created_at)
- `designs` (id, owner_user_id, product_id, variant_id, data jsonb, customer_comment, status:
  `DRAFT|SUBMITTED|LOCKED`, created_at)
- `production_files` (id, design_id, zone_code, file_id, width_px, height_px, dpi, width_cm, height_cm,
  effective_min_dpi, generated_at)
- `files` (id, storage_key, mime, size_bytes, sha256, created_at)

`designs.data` (JSON, versión 1):
```json
{
  "version": 1,
  "variantId": 123,
  "zones": [
    {
      "zoneCode": "FRONT",
      "widthCm": 30.0, "heightCm": 40.0,
      "layers": [
        { "id": "l1", "type": "image", "assetId": 55,
          "xCm": 5.0, "yCm": 3.0, "widthCm": 20.0, "heightCm": 25.0,
          "rotationDeg": 0, "z": 1, "flipX": false },
        { "id": "l2", "type": "text", "text": "Mauricio",
          "fontFamily": "Pacifico", "fontSizeCm": 2.5, "colorHex": "#FF6A00",
          "xCm": 8.0, "yCm": 30.0, "rotationDeg": 0, "z": 2 }
      ]
    }
  ]
}
```
- Coordenadas con origen en la esquina superior izquierda de la zona, en **cm**.
- **DPI efectivo** de una capa de imagen = `min(width_px / (widthCm/2.54), height_px / (heightCm/2.54))`.
  - ≥ 300: ok · 150–299: aviso amarillo · < 150: aviso rojo (se permite continuar con confirmación;
    el productor ve la alerta). Umbrales configurables.
- El editor 2D y el visor 3D leen y escriben **este mismo JSON**. El 3D nunca es fuente de verdad.

### 5.5 Pedidos
- `carts` (id, user_id, updated_at) + `cart_items` (cart_id, variant_id, design_id nullable, qty, comment)
- `orders` (id, number único `CL-2026-000123`, customer_id, status, subtotal, shipping_cost, gateway_fee,
  total, estimated_cost, real_cost, pricing_formula_version_id, payment_method, shipping_address jsonb,
  assigned_to nullable, created_by, customer_notes, created_at)
- `order_items` (id, order_id, variant_id, design_id nullable, qty, unit_price, price_breakdown jsonb,
  estimated_unit_cost, customer_comment)
- `order_status_history` (id, order_id, from_status, to_status, user_id, comment, created_at)

Estados: `PENDIENTE_PAGO → PAGADO → EN_PRODUCCION → LISTO → ENVIADO → ENTREGADO`, y `CANCELADO` desde
`PENDIENTE_PAGO` o `PAGADO` (con devolución de stock). Transiciones validadas en el backend (máquina de estados).

Reglas de stock:
- Al crear el pedido se **verifica** disponibilidad.
- Al pasar a `PAGADO` se **descuenta** la variante (movimiento `SALE`).
- Al pasar a `LISTO` se descuentan los insumos estimados (o los reales si el productor los ajustó).
- Al cancelar se devuelve lo descontado (movimiento `RETURN`).

### 5.6 Pagos y facturación
- `payments` (id, order_id, method: `TRANSFERENCIA|NEQUI|EFECTIVO|WOMPI`, amount, status:
  `PENDIENTE|CONFIRMADO|RECHAZADO`, reference, proof_file_id, confirmed_by, confirmed_at, destination:
  `CAJA|BANCO`)
  - Pago manual: el cliente sube comprobante; un empleado confirma → pedido pasa a `PAGADO`.
  - Se permite pago parcial (abonos): el pedido pasa a `PAGADO` cuando la suma confirmada ≥ total
    (configurable: % mínimo de anticipo para empezar producción).
- `PaymentGateway` (interfaz) con `ManualPaymentGateway` (activo) y `WompiPaymentGateway`
  (implementado contra la API de Wompi, apagado con `payments.wompi.enabled=false`; webhook con validación
  de firma).
- `invoices` (id, order_id, number consecutivo `FI-000001`, issued_at, snapshot jsonb, pdf_file_id, provider:
  `INTERNAL`)
- `quotes` (id, customer_id nullable, customer_name, items jsonb, total, valid_until, status:
  `BORRADOR|ENVIADA|ACEPTADA|VENCIDA`, converted_order_id) — cotizaciones manuales desde el panel.

### 5.7 Producción e incidentes
- Asignación: `orders.assigned_to`. "Tomar pedido" = autoasignarse. Solo el asignado o un admin avanzan el estado
  de producción.
- `incidents` (id, order_id, order_item_id, type: `ERROR_IMPRESION|ERROR_ESTAMPADO|PRODUCTO_DEFECTUOSO|OTRO`,
  description, reported_by, created_at, loss_amount bigint) + `incident_items` (incident_id, item_type, item_id,
  qty, unit_cost)
  - Registrar un incidente crea movimientos `INCIDENT` y suma la pérdida al `real_cost` del pedido.
- Vista de producción por ítem: PNG por zona (descargar), técnica, parámetros (temp/tiempo/presión), talla,
  color, comentario del cliente, alertas de DPI.

### 5.8 Envíos
- `carriers` (id, name, tracking_url_template, active) — p. ej. `https://.../rastreo?guia={tracking}`
- `shipments` (id, order_id único, carrier_id, tracking_number, cost bigint, charged_to_customer bigint,
  status: `PREPARANDO|DESPACHADO|ENTREGADO|DEVUELTO`, shipped_at, delivered_at)
- Por ahora el costo de envío lo pone el empleado o una tarifa fija por ciudad (`shipping_rates`: city, price).
  Recogida en el local = costo 0.

### 5.9 Finanzas
- `expense_categories` (id, name: Luz, Internet, Hosting/Dominio, Arriendo, Compra a proveedor, Publicidad,
  Mantenimiento equipos, Otros)
- `expenses` (id, category_id, date, amount, supplier_id nullable, description, receipt_file_id, user_id)
- Reportes (calculados, no tablas):
  - **Mensual:** ingresos (pagos confirmados) − (costo real de pedidos + gastos del mes + pérdidas por
    incidentes) = utilidad real; comparado contra utilidad estimada.
  - **Por producto:** unidades, ingresos, costo real, margen real.
  - **Por pedido:** precio vs. costo estimado vs. costo real.
  - Exportar a CSV/Excel.

### 5.10 Precios (fórmula configurable y versionada)
- `pricing_formulas` (id, name, version int, status: `BORRADOR|ACTIVA|ARCHIVADA`, rounding: `NONE|DOWN_1000|
  UP_1000|NEAREST_100`, created_by, activated_at)
- `pricing_components` (id, formula_id, code, label, sort_order, kind, value numeric, applies_to_technique
  nullable, applies_to_category nullable, stage: `COST|MARKUP|FEE`)
  - `kind`:
    - `VARIANT_PURCHASE_COST` — costo de compra de la variante
    - `LABOR` — `value` = valor hora; usa minutos de diseño/fabricación del producto (campos del producto)
    - `FIXED_PER_UNIT` — monto fijo (p. ej. empaque, envío a taller)
    - `PRINT_DTF_CM2` — área impresa con margen × costo por cm² (derivado de la config de DTF)
    - `PRINT_SUBLIMATION_CM2` — área × costo por cm² de papel+tinta
    - `SUPPLIES_ESTIMATED` — suma de `supply_consumption_rules`
    - `OVERHEAD_PER_ORDER` — gastos fijos mensuales / pedidos estimados del mes
    - `PERCENT_OF_COST` — imprevistos, administración, utilidad (porcentaje sobre el costo acumulado)
    - `PER_EXTRA_ELEMENT` — recargo por cada capa/zona adicional
    - `GATEWAY_FEE` — comisión de pasarela con **gross-up**: `precio = (neto + fijo) / (1 − pct)`
      (+ IVA sobre la comisión si aplica). Solo cuando el método es WOMPI.
- Productos: campos `design_minutes`, `production_minutes` y overrides de porcentajes por producto
  (`product_pricing_overrides`: product_id, component_code, value), porque el Excel usa porcentajes distintos
  por producto.
- Cada `order_item` guarda `price_breakdown` (componente → valor) y la versión de fórmula. Cambiar la fórmula
  nunca altera pedidos viejos.
- Endpoint de **simulación**: dado un producto/variante/diseño, devuelve precio y desglose con una fórmula
  (borrador o activa) para probar antes de activarla.

**Fórmula v1 (semilla, migrada del Excel actual — prueba de concepto, se va a reformular):**
```
costo = compra + valor_hora × (h_diseño + h_fabricación) + envío_taller
precio = ROUNDDOWN_1000( costo × (1.10 + imprevistos + administración + utilidad) )
```
con valor_hora = 8.750, y porcentajes por producto del Excel. El 1.10 se carga como componente editable
"Factor base" (su significado está pendiente de confirmar; ver `memory/decisions/`). Sobre v1 se suman
como componentes opcionales (inicialmente en 0 o desactivados): impresión DTF por cm², insumos, overhead,
comisión Wompi.

**Costo DTF de referencia:** metro de 58 × 100 cm = 5.800 cm² a $23.000 → **$3,97/cm²**.
Área cobrada por diseño = `(ancho + 2·margen) × (alto + 2·margen)`, margen por defecto 1 cm.

### 5.11 Notificaciones y auditoría
- `notifications` (id, user_id, type, title, body, link, read_at, created_at)
  - Eventos: pedido nuevo (a empleados), pedido asignado, pago por confirmar, stock bajo (a admins), cambio de
    estado (al cliente), incidente registrado (a admins).
  - Entrega in-app con polling cada 30 s o SSE. Interfaz `NotificationChannel` para correo/WhatsApp a futuro.
- `audit_log` (id, user_id, entity, entity_id, action, diff jsonb, created_at) — para pedidos, pagos, precios,
  stock.

## 6. Flujos principales

**Cliente personaliza y compra**
1. Entra al catálogo → elige producto → elige color y talla (ve guía de tallas).
2. Abre el personalizador: sube imágenes / agrega texto por zona; mueve, escala, rota; alterna vista 2D/3D.
3. El sistema muestra el DPI efectivo y avisa si la imagen se ve pixelada.
4. Escribe un comentario opcional → "Agregar al carrito" (se guarda el diseño `SUBMITTED`).
5. Checkout: dirección o recogida, método de pago → se crea el pedido `PENDIENTE_PAGO` con precio congelado.
6. Pago manual: ve los datos de cuenta/Nequi, sube comprobante. Un empleado confirma → `PAGADO`.
7. Sigue el estado en "Mis pedidos" y ve la guía cuando se despacha.

**Producción**
1. Llega notificación "pedido nuevo". El empleado lo toma (queda asignado).
2. Ve cada ítem con sus PNG por zona (generados al confirmar el pedido), técnica y parámetros.
3. Descarga e imprime. Si algo falla, registra incidente (descuenta stock y suma pérdida).
4. Marca `LISTO` (se descuentan insumos), crea el envío con guía → `ENVIADO` → `ENTREGADO`.
5. Al pasar a `PAGADO` se emite la factura interna.

**Admin**
- Gestiona catálogo, variantes, zonas y medidas, parámetros de técnica, insumos, proveedores, compras,
  empleados, transportadoras, gastos, fórmulas de precio, y ve reportes.

## 7. Generación del PNG de producción

- Se dispara al confirmar el pedido (job asíncrono con estado en `production_files`; reintentable).
- Por zona: lienzo de `widthCm/2.54 × 300` × `heightCm/2.54 × 300` píxeles, transparente; se pintan las capas
  en orden `z` con su posición, tamaño, rotación y espejo; textos con las mismas fuentes que el editor
  (las fuentes `.ttf` viven en el backend y en el frontend).
- Si una capa tiene DPI efectivo < 300 y el upscaler está disponible, se usa la versión mejorada del asset.
- Guardar con metadatos de 300 DPI (chunk pHYs). Nombre: `CL-2026-000123_item2_FRONT_30x40cm.png`.
- **Prueba obligatoria:** un diseño conocido produce un PNG con dimensiones exactas y la capa en la posición
  esperada (tolerancia ±1 px).
- Opción "recortar al contenido" para DTF (ahorra film): genera además un PNG recortado al bounding box de las
  capas con su posición anotada.

## 8. Upscaler (workers/upscaler)

- Servicio pequeño (FastAPI) con `POST /upscale {fileUrl|path, scale}` que ejecuta
  `realesrgan-ncnn-vulkan` (usa la GPU disponible; si no hay, responde 503 y el backend sigue sin mejora).
- El backend lo llama de forma asíncrona y guarda `upscaled_file_id`. Nunca bloquea el checkout.

## 9. UX y diseño visual

- **Tienda (cliente):** fondo oscuro (#0E0E10 aprox.), acentos con el degradado del logo
  (rojo #E53935 → naranja #FF8A00 → amarillo #FFD400 → verde #3DDC84), tipografía display tipo script para
  títulos de marca y una sans legible para el resto. Mobile-first, botones grandes, el editor usable con el dedo
  (pinch para escalar, dos dedos para rotar).
- **Panel de producción/admin:** tema claro, denso y legible, tablas con filtros, pensado para trabajar horas.
- Logo en `assets/logo.png`.
- Accesibilidad básica: contraste AA, foco visible, textos alternativos.

## 10. Seguridad
- Contraseñas con BCrypt. JWT de acceso corto (15 min) + refresh rotativo.
- Autorización por rol en cada endpoint (`/api/admin/**` → ADMIN/EMPLEADO según recurso).
- Un cliente solo ve sus pedidos, diseños y archivos. Las URLs de archivos son firmadas o pasan por un endpoint
  que valida permisos.
- Validar tipo y tamaño de archivos subidos (PNG, JPG, WEBP, SVG sanitizado, PDF; máx. 25 MB), recalcular
  dimensiones en el servidor.
- Rate limit en login y subida de archivos.
- Secretos solo por variables de entorno.

## 11. Fases (ver prompts/)
0. Fundación (1 agente)
1. Ola 1 en paralelo: catálogo+inventario · precios+finanzas · editor 2D · tienda cliente
2. Ola 2 en paralelo: pedidos+pagos+producción backend · panel admin/producción · visor 3D · upscaler+render
3. Ola 3: integración + E2E · despliegue

## 12. Propiedad de carpetas (para trabajar en paralelo sin pisarse)

| Agente | Es dueño de |
|---|---|
| A0 Fundación | todo el esqueleto, `common/`, `auth/`, `docs/openapi.yaml` (versión inicial), `infra/` inicial |
| A1 Catálogo e inventario | `backend/.../catalog/`, `backend/.../inventory/` |
| A2 Precios y finanzas | `backend/.../pricing/`, `backend/.../finance/`, `backend/.../billing/` (cotizaciones, facturas internas, `InvoiceProvider`) |
| A3 Editor 2D | `frontend/src/features/editor2d/`, `frontend/src/shared/design/` (tipos y utilidades del JSON de diseño) |
| A4 Tienda cliente | `frontend/src/features/store/`, `features/account/`, `features/auth/` (UI) |
| B1 Pedidos y producción (backend) | `backend/.../orders/`, `payments/`, `production/`, `shipping/`, `notifications/` (llama a `billing` para emitir la factura) |
| B2 Panel admin | `frontend/src/features/admin/` |
| B3 Visor 3D | `frontend/src/features/viewer3d/`, `frontend/public/models/` |
| B4 Render y upscaler | `backend/.../design/`, `workers/upscaler/`, `docs/design-schema.json`, `backend/src/main/resources/fonts/` |
| C1 Integración y QA | `frontend/e2e/`, arreglos transversales coordinados |
| C2 Despliegue | `infra/` |

Archivos compartidos (`docs/openapi.yaml`, `frontend/src/shared/api/`, `frontend/src/app/router.tsx`,
`application.yml`) se tocan siguiendo las reglas de `CLAUDE.md` §Coordinación.

Migraciones Flyway: la Fundación crea **todo el esquema** en `V1__schema.sql` y semillas en `V2__seed.sql`.
Si un agente necesita otra migración, usa **su rango**: A1 `V100–V199`, A2 `V200–V299`, B1 `V300–V399`,
B4 `V400–V499`, C1 `V500–V599`. `spring.flyway.out-of-order=true` en dev.
