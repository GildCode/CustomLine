# Custom Line — Especificación del sistema (v2)

**Versión:** 2 · **Fecha:** 2026-11-10 · **Reemplaza a:** la especificación v1 (guardada aparte como referencia histórica).
**Motivo del cambio:** ver `memory/decisions/2026-11-10-spec-v2.md`.

Fuente de verdad del proyecto. Si un prompt o una decisión contradice este archivo, gana este archivo, salvo que un archivo de `memory/decisions/` registre el cambio con fecha y motivo.

El modelo de datos de la §5 corresponde al diagrama `docs/customDB_boceto_v2.drawio`. Si el diagrama y este archivo no coinciden, se corrigen los dos en el mismo cambio.

---

## 1. Qué es

Custom Line es un negocio de personalización de prendas y artículos (camisetas, buzos, chompas, mugs, termos, carcasas de celular, mouse pads, llaveros, etc.) que trabaja con dos técnicas:

- **Sublimación:** solo en prendas blancas con ≥ 70 % poliéster y en artículos con recubrimiento de poliéster (mugs, carcasas, mouse pads, termos…). Se imprime con tinta de sublimación, se fija con cinta y se aplica calor con presión. Cada producto tiene parámetros propios (temperatura, tiempo, presión).
- **DTF (Direct to Film):** solo prendas; sirve en casi cualquier tela apta al calor. Proceso estándar (misma temperatura y tiempo), pero se guardan parámetros por si cambia.

La plataforma es una tienda online responsive con dos caras:

- **Cliente:** catálogo, personalizador 2D + 3D, carrito, pedidos, cuenta, historial y estado de pedidos.
- **Producción / administración:** dashboard de pedidos, archivos listos para imprimir, inventario de productos e insumos, proveedores, precios, facturación interna, pagos, gastos, reportes de ganancias, incidentes y envíos.

El núcleo del negocio es la personalización. También se venden **artículos terminados**: comprados ya hechos o producidos por el taller para tener en stock.

### Principio rector de finanzas

La prueba de concepto en Excel calculaba precios pero "no se vio la plata". El sistema debe responder siempre: **¿cuánto entró, cuánto salió y cuánto quedó, por pedido, por producto y por mes?** Por eso:

- Se congela el precio y el costo estimado al crear el pedido.
- Se registra el costo real al producir (insumos, reprocesos, incidentes, envío).
- Se registran todos los gastos de operación.
- Se calculan dos números distintos: **utilidad** y **flujo de caja** (§5, reglas de finanzas).

## 2. Alcance

### Incluido (MVP)

- Cuentas de usuario con roles fijos (ADMIN, EMPLEADO, CLIENTE); un usuario puede tener varios.
- Clientes sin acceso (sin contraseña) creados por un empleado para pedidos por WhatsApp o en el local.
- Catálogo con productos, variantes (talla/color/material), tallas como catálogo, zonas imprimibles con medidas en cm por talla.
- Personalizador 2D (fuente de verdad del diseño) y visor 3D (vista previa del mismo diseño).
- Carga de imágenes y textos por zona; comentario libre del cliente por ítem.
- Validación de resolución (DPI efectivo) con aviso al cliente y alerta al productor.
- Generación automática de PNG a tamaño real, 300 DPI, fondo transparente, por zona, listo para imprimir.
- Carrito persistente en la base de datos, con productos personalizados y terminados.
- Pedido como unidad (un estado, un envío). Reserva de stock al crear el pedido y vencimiento de pedidos no pagados.
- Pagos manuales (transferencia, Nequi, efectivo) confirmados por un empleado, con abonos.
- Pasarela Wompi **implementada y apagada**: tablas, interfaz `PaymentGateway`, webhooks idempotentes con validación de firma. Se activa por configuración.
- Reembolsos.
- Factura interna con consecutivo propio, detalle por línea, IVA y PDF; anulación con nota crédito. Estructura lista para factura electrónica DIAN (`InvoiceProvider`).
- Inventario de variantes e insumos con movimientos con motivo; alertas de stock bajo.
- Producción para stock de artículos terminados (movimiento PRODUCCION).
- Proveedores, compras (como movimientos) y gastos de operación.
- Precios por fórmula con variables editables desde el panel, ajustes por producto e historial de cambios.
- Producción: asignación de pedido a un empleado, parámetros de técnica visibles, registro de quién hizo qué.
- Incidentes/reprocesos: descuentan stock y registran pérdida.
- Envíos: transportadora (texto), guía, costo, estado.
- Notificaciones solo dentro de la plataforma.
- Reportes: utilidad y flujo de caja por mes, margen por pedido (estimado vs. real), margen estimado por producto, stock bajo. Exportar a CSV/Excel.
- Archivos en Cloudflare R2 (bucket privado, URLs firmadas).
- Anonimización de clientes (Ley 1581 de Habeas Data).
- Despliegue en un PC propio con Docker Compose + Cloudflare Tunnel + backups automáticos fuera del PC.

### Fuera de alcance por ahora (dejar extensible, no construir)

- Factura electrónica DIAN (solo interfaz y columnas: `cufe`, XML, estados DIAN).
- Wompi en producción (implementado pero apagado).
- Upscaler con Real-ESRGAN. Cuando se haga, se agrega `uid_archivo_mejorado` a `archivos`.
- Cotizaciones formales (quotes).
- Tabla de transportadoras y tarifas de envío por ciudad (hoy: transportadora en texto, costo lo pone el empleado).
- Reglas automáticas de consumo de insumos (hoy: el empleado registra el consumo, con valores sugeridos).
- Auditoría general (`audit_log`). Hoy se audita con `pedidos_historial`, `variables_historial`, `movimientos` y los campos `uid_confirmado_por`, `uid_creado_por`, etc.
- Costo real por línea de pedido (hoy: real por pedido, estimado por línea).
- Varios envíos por pedido.
- Corte láser, toldos y otros negocios del Excel.
- App nativa / Flutter.
- Notificaciones por correo o WhatsApp (dejar `NotificationChannel` extensible).
- Producción por etapas o tercerizada (hoy: un pedido → un responsable).

## 3. Stack

| Capa | Tecnología |
| --- | --- |
| Backend | Java 21, Spring Boot 3.x, Spring Web, Spring Security (JWT access + refresh rotativo), Spring Data JPA, Bean Validation, Flyway, springdoc-openapi, MapStruct, Lombok opcional |
| Base de datos | PostgreSQL 16 (o superior). Extensión `citext`. |
| IDs | UUID v7 generados en la aplicación para entidades principales; SERIAL/BIGSERIAL para catálogos internos y registros |
| Tests backend | JUnit 5, Testcontainers (Postgres real), MockMvc / RestAssured |
| Frontend | React 18 + TypeScript + Vite, React Router, TanStack Query, Zustand, Tailwind CSS, react-konva (editor 2D), @react-three/fiber + @react-three/drei (3D), MSW (mocks), Vitest + Testing Library, Playwright (E2E) |
| Imágenes | Render de PNG de producción en el backend (Java2D/ImageIO a 300 DPI con metadatos pHYs) |
| Archivos | Cloudflare R2 (S3 compatible, SDK de AWS S3 con endpoint de R2), bucket privado, detrás de la interfaz `AlmacenamientoArchivos` |
| Infra | Docker Compose, Nginx (sirve el frontend y hace proxy a /api), Cloudflare Tunnel, pg_dump + rclone para backups (base de datos y bucket) |

- **Moneda:** COP en enteros (`BIGINT`, pesos sin decimales). Precios unitarios sin IVA.
- **Zona horaria:** America/Bogota; columnas `TIMESTAMPTZ`.
- **Idioma:** UI en español; tablas y columnas en español (snake_case).
- **Medidas físicas:** centímetros, `NUMERIC(6,2)`.
- **Temperaturas:** °F (así las usa el taller).

## 4. Estructura del repositorio (monorepo)

```
custom-line/
├── CLAUDE.md                  # reglas para todos los agentes (lo carga Claude Code)
├── docs/
│   ├── SPEC.md                # este archivo (v2)
│   ├── customDB_boceto_v2.drawio  # diagrama del modelo de datos
│   └── openapi.yaml           # contrato de la API (lo crea la Fundación; dueño: ver §12)
├── memory/                    # memoria persistente de los agentes (ver CLAUDE.md)
│   ├── PROGRESS.md            # generado por scripts/progress.sh
│   ├── decisions/             # un archivo por decisión
│   ├── requests/              # peticiones entre agentes, un archivo cada una
│   └── agents/<agente>.md     # memoria de cada agente
├── scripts/progress.sh
├── backend/                   # Spring Boot
│   └── src/main/java/co/customline/
│       ├── common/            # errores, paginación, money, UUID v7, config, almacenamiento de archivos
│       ├── auth/              # usuarios, roles, JWT, refresh tokens, anonimización
│       ├── catalog/           # categorías, productos, variantes, tallas, zonas, técnicas, parámetros, proveedores
│       ├── inventory/         # insumos, movimientos, alertas de stock, producción para stock
│       ├── pricing/           # variables globales, ajustes, historial, cálculo y simulación de precio
│       ├── design/            # diseños, bloqueo, render PNG, validación DPI
│       ├── orders/            # carrito, pedidos, máquina de estados, historial, vencimiento
│       ├── payments/          # métodos, pagos, abonos, reembolsos, PaymentGateway (Wompi), webhooks
│       ├── billing/           # resoluciones, facturas, notas crédito, datos de facturación, InvoiceProvider
│       ├── production/        # asignación, incidentes, costo real
│       ├── shipping/          # envíos
│       ├── finance/           # gastos, reportes
│       └── notifications/     # notificaciones in-app
├── frontend/
│   └── src/
│       ├── app/               # router, providers, layout
│       ├── shared/            # api client, ui kit, hooks, tipos generados
│       ├── features/
│       │   ├── auth/
│       │   ├── store/         # catálogo, producto, carrito, checkout (cliente)
│       │   ├── account/       # mis pedidos, perfil, direcciones, datos de facturación
│       │   ├── editor2d/      # personalizador 2D
│       │   ├── viewer3d/      # visor 3D
│       │   └── admin/         # producción, inventario, precios, pagos, facturación, finanzas
│       └── mocks/             # MSW handlers
└── infra/                     # docker-compose, nginx, cloudflared, backups
```

## 5. Modelo de datos

**El esquema SQL (`V1__schema.sql`) lo escribe el dueño del proyecto (Andrés) a partir de esta sección.** Los agentes no lo generan ni lo modifican; si necesitan un cambio, lo piden en `memory/requests/` y el cambio entra como migración nueva en su rango (§12). Prohibido `spring.jpa.hibernate.ddl-auto` distinto de `validate`.

Todas las restricciones listadas aquí (CHECK, UQ, FK compuestas, ON DELETE, índices) son obligatorias en el SQL. Las reglas de negocio son obligatorias en el backend y cada una tiene su test.

### 5.1 Tablas

Formato: `columna` tipo. **PK** = llave primaria (varias en una tabla = compuesta). `→ tabla` = FK. `UQ` = único. `NULL` = admite nulos; lo demás es `NOT NULL`.

#### Acceso y usuarios

- **roles**: `id_rol` SERIAL **PK**, `codigo` VARCHAR(20) UQ, `nombre` VARCHAR(50)
- **usuarios_roles**: `uid_usuario` UUID **PK** → usuarios, `id_rol` INT **PK** → roles, `created_at` TIMESTAMPTZ
- **usuarios**: `uid_usuario` UUID **PK**, `nombre` VARCHAR(80), `apellido_1` VARCHAR(80), `apellido_2` VARCHAR(80) NULL, `email` CITEXT UQ NULL, `telefono` VARCHAR(20) NULL, `password_hash` VARCHAR(255) NULL, `activo` BOOLEAN, `ultimo_acceso` TIMESTAMPTZ NULL, `anonimizado_en` TIMESTAMPTZ NULL, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **direcciones**: `uid_direccion` UUID **PK**, `uid_usuario` UUID → usuarios, `etiqueta` VARCHAR(40), `departamento` VARCHAR(60), `ciudad` VARCHAR(60), `direccion` VARCHAR(150), `telefono` VARCHAR(20), `notas` VARCHAR(200), `predeterminada` BOOLEAN, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **refresh_tokens**: `uid_token` UUID **PK**, `uid_usuario` UUID → usuarios, `uid_familia` UUID, `token_hash` VARCHAR(255) UQ, `expira_en` TIMESTAMPTZ, `revocado_en` TIMESTAMPTZ NULL, `uid_reemplazado_por` UUID NULL → refresh_tokens, `ip` VARCHAR(45), `user_agent` VARCHAR(255), `created_at` TIMESTAMPTZ

#### Archivos (nube R2)

- **archivos**: `uid_archivo` UUID **PK**, `uid_usuario` UUID NULL → usuarios, `proveedor` VARCHAR(20), `bucket` VARCHAR(60), `storage_key` VARCHAR(255) UQ, `nombre_original` VARCHAR(200), `mime` VARCHAR(80), `tamano_bytes` BIGINT, `ancho_px` INT NULL, `alto_px` INT NULL, `sha256` CHAR(64), `estado` VARCHAR(15), `eliminar_despues_de` TIMESTAMPTZ NULL, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ

#### Notificaciones

- **notificaciones**: `id_notificacion` BIGSERIAL **PK**, `uid_usuario` UUID → usuarios, `tipo` VARCHAR(40), `titulo` VARCHAR(120), `cuerpo` TEXT, `link` VARCHAR(255), `leida_en` TIMESTAMPTZ NULL, `created_at` TIMESTAMPTZ

#### Gastos

- **categorias_gasto**: `id_categoria_gasto` SERIAL **PK**, `nombre` VARCHAR(60) UQ, `activo` BOOLEAN
- **gastos**: `uid_gasto` UUID **PK**, `id_categoria_gasto` INT → categorias_gasto, `uid_proveedor` UUID NULL → proveedores, `fecha` DATE, `monto` BIGINT (COP), `descripcion` VARCHAR(200), `uid_archivo_soporte` UUID NULL → archivos, `uid_usuario` UUID → usuarios, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ

#### Catálogo

- **categorias**: `id_categoria` SERIAL **PK**, `id_categoria_padre` INT NULL → categorias, `nombre` VARCHAR(60), `slug` VARCHAR(80) UQ, `orden` SMALLINT, `activo` BOOLEAN
- **productos**: `uid_producto` UUID **PK**, `id_categoria` INT → categorias, `nombre` VARCHAR(100), `slug` VARCHAR(120) UQ, `descripcion` TEXT, `tipo` VARCHAR(15), `tarifa_iva` NUMERIC(5,2), `minutos_diseno` INT, `minutos_fabricacion` INT, `uid_archivo_modelo3d` UUID NULL → archivos, `activo` BOOLEAN, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **producto_variantes**: `uid_variante` UUID **PK**, `uid_producto` UUID → productos, `id_talla` INT NULL → tallas, `uid_proveedor` UUID NULL → proveedores, `sku` VARCHAR(40) UQ, `color_nombre` VARCHAR(40), `color_hex` CHAR(7), `genero` VARCHAR(10) NULL, `material` VARCHAR(40) NULL, `porcentaje_poliester` SMALLINT NULL, `costo_compra` BIGINT (COP), `precio_venta` BIGINT (COP) NULL, `stock` INT, `stock_minimo` INT, `activo` BOOLEAN, `version` INT, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **proveedores**: `uid_proveedor` UUID **PK**, `nombre` VARCHAR(100), `contacto` VARCHAR(80), `telefono` VARCHAR(20), `whatsapp` VARCHAR(20), `correo` VARCHAR(120), `pagina_web` VARCHAR(200), `pais` VARCHAR(60), `ciudad` VARCHAR(60), `direccion` VARCHAR(150), `activo` BOOLEAN, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **producto_imagenes**: `id_imagen` SERIAL **PK**, `uid_producto` UUID → productos, `uid_archivo` UUID → archivos, `orden` SMALLINT
- **zonas_medidas**: `id_zona_medida` SERIAL **PK**, `id_zona` INT → zonas_impresion, `id_talla` INT NULL → tallas, `ancho_max_cm` NUMERIC(6,2), `alto_max_cm` NUMERIC(6,2)
- **tallas**: `id_talla` SERIAL **PK**, `codigo` VARCHAR(10) UQ, `nombre` VARCHAR(30), `orden` SMALLINT
- **zonas_impresion**: `id_zona` SERIAL **PK**, `uid_producto` UUID → productos, `codigo` VARCHAR(20), `nombre` VARCHAR(40), `uv_rect` JSONB NULL, `uid_archivo_overlay` UUID NULL → archivos
- **ajustes_precio_producto**: `id_ajuste` SERIAL **PK**, `uid_producto` UUID → productos, `clave` VARCHAR(50), `valor` NUMERIC(14,4), `updated_at` TIMESTAMPTZ
- **producto_tecnicas**: `uid_producto` UUID **PK** → productos, `id_tecnica` INT **PK** → tecnicas
- **tecnicas**: `id_tecnica` SERIAL **PK**, `codigo` VARCHAR(20) UQ, `nombre` VARCHAR(50), `config` JSONB, `activo` BOOLEAN, `updated_at` TIMESTAMPTZ
- **parametros_tecnica**: `id_parametro` SERIAL **PK**, `id_tecnica` INT → tecnicas, `uid_producto` UUID NULL → productos, `temperatura_f` SMALLINT, `tiempo_seg` SMALLINT, `presion` VARCHAR(20), `notas` VARCHAR(200), `updated_at` TIMESTAMPTZ

#### Configuración de precios

- **variables_globales**: `id_variable` SERIAL **PK**, `clave` VARCHAR(50) UQ, `valor` NUMERIC(14,4), `tipo` VARCHAR(15), `descripcion` VARCHAR(150), `uid_actualizado_por` UUID → usuarios, `updated_at` TIMESTAMPTZ
- **variables_historial**: `id_historial` BIGSERIAL **PK**, `id_variable` INT NULL → variables_globales, `id_ajuste` INT NULL → ajustes_precio_producto, `clave` VARCHAR(50), `valor_anterior` NUMERIC(14,4) NULL, `valor_nuevo` NUMERIC(14,4) NULL, `uid_usuario` UUID → usuarios, `created_at` TIMESTAMPTZ

#### Diseño / personalización

- **disenos**: `uid_diseno` UUID **PK**, `uid_usuario` UUID → usuarios, `uid_variante` UUID → producto_variantes, `uid_diseno_origen` UUID NULL → disenos, `data` JSONB, `version_esquema` SMALLINT, `comentario` VARCHAR(500), `estado` VARCHAR(15), `bloqueado_en` TIMESTAMPTZ NULL, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **disenos_archivos**: `uid_diseno` UUID **PK** → disenos, `uid_archivo` UUID **PK** → archivos
- **archivos_produccion**: `uid_archivo_produccion` UUID **PK**, `uid_detalle_pedido` UUID → detalle_pedido, `id_zona` INT → zonas_impresion, `uid_archivo` UUID NULL → archivos, `ancho_cm` NUMERIC(6,2), `alto_cm` NUMERIC(6,2), `dpi` SMALLINT, `dpi_efectivo_min` SMALLINT, `estado` VARCHAR(15), `intentos` SMALLINT, `error` TEXT NULL, `generado_en` TIMESTAMPTZ NULL, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ

#### Pedidos

- **detalle_pedido**: `uid_detalle_pedido` UUID **PK**, `uid_pedido` UUID → pedidos, `uid_variante` UUID → producto_variantes, `uid_diseno` UUID NULL → disenos, `id_tecnica` INT NULL → tecnicas, `cantidad` INT, `precio_unitario` BIGINT (COP), `tarifa_iva` NUMERIC(5,2), `costo_unitario_estimado` BIGINT (COP), `desglose_precio` JSONB, `comentario` VARCHAR(500), `created_at` TIMESTAMPTZ
- **carritos**: `uid_carrito` UUID **PK**, `uid_usuario` UUID UQ → usuarios, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **carrito_items**: `uid_carrito_item` UUID **PK**, `uid_carrito` UUID → carritos, `uid_variante` UUID → producto_variantes, `uid_diseno` UUID NULL → disenos, `id_tecnica` INT NULL → tecnicas, `cantidad` INT, `comentario` VARCHAR(500), `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **pedidos**: `uid_pedido` UUID **PK**, `numero` VARCHAR(20) UQ, `uid_cliente` UUID → usuarios, `canal` VARCHAR(10), `estado` VARCHAR(20), `subtotal` BIGINT (COP), `impuestos` BIGINT (COP), `descuento` BIGINT (COP), `costo_envio` BIGINT (COP), `total` BIGINT (COP), `total_pagado` BIGINT (COP), `costo_estimado` BIGINT (COP), `costo_real` BIGINT (COP), `tipo_entrega` VARCHAR(10), `direccion_envio` JSONB NULL, `notas_cliente` VARCHAR(500), `expira_en` TIMESTAMPTZ NULL, `uid_asignado_a` UUID NULL → usuarios, `uid_creado_por` UUID → usuarios, `version` INT, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **pedidos_historial**: `id_historial` BIGSERIAL **PK**, `uid_pedido` UUID → pedidos, `estado_anterior` VARCHAR(20) NULL, `estado_nuevo` VARCHAR(20), `uid_usuario` UUID → usuarios, `comentario` VARCHAR(300), `created_at` TIMESTAMPTZ

#### Producción y envíos

- **incidentes**: `uid_incidente` UUID **PK**, `uid_pedido` UUID → pedidos, `uid_detalle_pedido` UUID NULL → detalle_pedido, `tipo` VARCHAR(30), `descripcion` TEXT, `perdida_adicional` BIGINT (COP), `uid_reportado_por` UUID → usuarios, `created_at` TIMESTAMPTZ
- **envios**: `uid_envio` UUID **PK**, `uid_pedido` UUID UQ → pedidos, `transportadora` VARCHAR(60), `numero_guia` VARCHAR(60), `costo` BIGINT (COP), `estado` VARCHAR(15), `despachado_en` TIMESTAMPTZ NULL, `entregado_en` TIMESTAMPTZ NULL, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ

#### Inventario

- **movimientos**: `uid_movimiento` UUID **PK**, `motivo` VARCHAR(15), `uid_pedido` UUID NULL → pedidos, `uid_incidente` UUID NULL → incidentes, `uid_proveedor` UUID NULL → proveedores, `nota` VARCHAR(300), `uid_usuario` UUID → usuarios, `created_at` TIMESTAMPTZ
- **detalle_movimiento**: `id_detalle_movimiento` BIGSERIAL **PK**, `uid_movimiento` UUID → movimientos, `uid_variante` UUID NULL → producto_variantes, `id_insumo` INT NULL → insumos, `cantidad` NUMERIC(10,2) ±, `costo_unitario` BIGINT (COP)
- **insumos**: `id_insumo` SERIAL **PK**, `uid_proveedor` UUID NULL → proveedores, `nombre` VARCHAR(100), `unidad` VARCHAR(10), `costo_unitario` BIGINT (COP), `stock` NUMERIC(10,2), `stock_minimo` NUMERIC(10,2), `activo` BOOLEAN, `version` INT, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ

#### Pagos (manual + pasarela lista, apagada)

- **pasarelas**: `id_pasarela` SERIAL **PK**, `codigo` VARCHAR(20) UQ, `nombre` VARCHAR(50), `ambiente` VARCHAR(12), `activo` BOOLEAN, `config_publica` JSONB, `updated_at` TIMESTAMPTZ
- **pagos_eventos**: `id_evento` BIGSERIAL **PK**, `id_pasarela` INT → pasarelas, `uid_pago` UUID NULL → pagos, `referencia_evento` VARCHAR(100), `tipo_evento` VARCHAR(60), `payload` JSONB, `firma_valida` BOOLEAN, `procesado_en` TIMESTAMPTZ NULL, `error` TEXT NULL, `recibido_en` TIMESTAMPTZ
- **metodos_pago**: `id_metodo` SERIAL **PK**, `id_pasarela` INT NULL → pasarelas, `codigo` VARCHAR(30) UQ, `nombre` VARCHAR(60), `tipo` VARCHAR(10), `comision_pct` NUMERIC(6,4), `comision_fija` BIGINT (COP), `trasladar_comision` BOOLEAN, `instrucciones` TEXT, `activo` BOOLEAN, `updated_at` TIMESTAMPTZ
- **pagos**: `uid_pago` UUID **PK**, `uid_pedido` UUID → pedidos, `id_metodo` INT → metodos_pago, `monto` BIGINT (COP), `recargo_cliente` BIGINT (COP), `comision` BIGINT (COP), `moneda` CHAR(3), `estado` VARCHAR(15), `referencia` VARCHAR(60) UQ, `clave_idempotencia` VARCHAR(80) UQ, `referencia_externa` VARCHAR(100) NULL, `respuesta_pasarela` JSONB NULL, `uid_archivo_comprobante` UUID NULL → archivos, `destino` VARCHAR(10), `uid_confirmado_por` UUID NULL → usuarios, `confirmado_en` TIMESTAMPTZ NULL, `expira_en` TIMESTAMPTZ NULL, `version` INT, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **reembolsos**: `uid_reembolso` UUID **PK**, `uid_pago` UUID → pagos, `monto` BIGINT (COP), `motivo` VARCHAR(300), `estado` VARCHAR(15), `referencia_externa` VARCHAR(100) NULL, `uid_usuario` UUID → usuarios, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ

#### Facturación (interna, lista para DIAN)

- **facturas**: `uid_factura` UUID **PK**, `uid_pedido` UUID → pedidos, `id_resolucion` INT → resoluciones_facturacion, `id_datos_facturacion` INT NULL → datos_facturacion, `prefijo` VARCHAR(10), `numero` BIGINT, `numero_completo` VARCHAR(30) UQ, `estado` VARCHAR(15), `cliente_snapshot` JSONB, `subtotal` BIGINT (COP), `impuestos` BIGINT (COP), `total` BIGINT (COP), `cufe` VARCHAR(120) NULL, `uid_archivo_pdf` UUID NULL → archivos, `uid_archivo_xml` UUID NULL → archivos, `emitida_en` TIMESTAMPTZ, `anulada_en` TIMESTAMPTZ NULL, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **detalle_factura**: `id_detalle_factura` BIGSERIAL **PK**, `uid_factura` UUID → facturas, `uid_detalle_pedido` UUID NULL → detalle_pedido, `codigo` VARCHAR(40), `descripcion` VARCHAR(200), `unidad_medida` VARCHAR(10), `cantidad` INT, `precio_unitario` BIGINT (COP), `descuento` BIGINT (COP), `iva_pct` NUMERIC(5,2), `iva_valor` BIGINT (COP), `total` BIGINT (COP)
- **notas_credito**: `uid_nota_credito` UUID **PK**, `uid_factura` UUID → facturas, `id_resolucion` INT → resoluciones_facturacion, `prefijo` VARCHAR(10), `numero` BIGINT, `numero_completo` VARCHAR(30) UQ, `motivo` VARCHAR(300), `subtotal` BIGINT (COP), `impuestos` BIGINT (COP), `total` BIGINT (COP), `estado` VARCHAR(15), `cufe` VARCHAR(120) NULL, `uid_archivo_pdf` UUID NULL → archivos, `emitida_en` TIMESTAMPTZ, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- **resoluciones_facturacion**: `id_resolucion` SERIAL **PK**, `tipo_documento` VARCHAR(15), `modalidad` VARCHAR(12), `prefijo` VARCHAR(10), `numero_resolucion` VARCHAR(40) NULL, `rango_desde` BIGINT, `rango_hasta` BIGINT, `consecutivo_actual` BIGINT, `vigente_desde` DATE, `vigente_hasta` DATE NULL, `activa` BOOLEAN, `version` INT, `updated_at` TIMESTAMPTZ
- **detalle_nota_credito**: `id_detalle_nota` BIGSERIAL **PK**, `uid_nota_credito` UUID → notas_credito, `id_detalle_factura` BIGINT → detalle_factura, `cantidad` INT, `precio_unitario` BIGINT (COP), `iva_pct` NUMERIC(5,2), `iva_valor` BIGINT (COP), `total` BIGINT (COP)
- **datos_facturacion**: `id_datos_facturacion` SERIAL **PK**, `uid_usuario` UUID → usuarios, `tipo_persona` VARCHAR(10), `tipo_documento` VARCHAR(10), `numero_documento` VARCHAR(20), `dv` CHAR(1) NULL, `razon_social` VARCHAR(150), `email_facturacion` VARCHAR(120), `direccion` VARCHAR(150), `ciudad` VARCHAR(60), `responsabilidad_fiscal` VARCHAR(20), `predeterminado` BOOLEAN, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ

### 5.2 Convenciones, índices, restricciones y reglas de negocio


#### Convenciones

- **UUID** (generado como **UUID v7** en la app, ordenado por tiempo): entidades principales y lo que se expone en la API.
- **SERIAL / BIGSERIAL**: catálogos internos, tablas puente y registros (logs).
- Dinero en **BIGINT COP** (pesos enteros). Precios unitarios SIN IVA. Medidas en cm NUMERIC(6,2). Temperatura en °F. Fechas TIMESTAMPTZ.
- `→ tabla` en la lista de columnas = FK con `REFERENCES` a esa tabla (en el diagrama algunas no tienen línea dibujada, pero existen igual).
- UQ = único · varias columnas **PK** en una tabla = llave compuesta · version = bloqueo optimista (@Version).
- Archivos: bucket privado R2; se guarda storage_key, nunca la URL. Subida y lectura con URLs firmadas.
- Pasarela: secretos solo en variables de entorno. Wompi registrado con activo = false.

#### Índices (además de PK y UQ)

- Toda columna FK lleva índice (Postgres no los crea solo).
- pedidos(uid_cliente, created_at) · pedidos(estado) · pagos(uid_pedido, estado) · movimientos(created_at)
- notificaciones(uid_usuario) WHERE leida_en IS NULL · archivos(sha256) · archivos(eliminar_despues_de) WHERE eliminar_despues_de IS NOT NULL
- gastos(fecha) · facturas(emitida_en) · disenos(uid_usuario, estado) · refresh_tokens(uid_familia)


#### Restricciones

- **CHECK estados**: pedidos PENDIENTE_PAGO|PAGADO|EN_PRODUCCION|LISTO|ENVIADO|ENTREGADO|CANCELADO · pagos PENDIENTE|APROBADO|RECHAZADO|ANULADO|ERROR · reembolsos PENDIENTE|COMPLETADO|FALLIDO · disenos BORRADOR|BLOQUEADO · archivos SUBIENDO|LISTO|RECHAZADO · archivos_produccion PENDIENTE|GENERANDO|LISTO|ERROR · facturas y notas EMITIDA|ANULADA|ENVIADA_DIAN|ACEPTADA_DIAN|RECHAZADA_DIAN · envios PREPARANDO|DESPACHADO|ENTREGADO|DEVUELTO · movimientos.motivo COMPRA|VENTA|CONSUMO|PRODUCCION|AJUSTE|INCIDENTE|DEVOLUCION · productos.tipo PERSONALIZABLE|TERMINADO · pedidos.canal WEB|WHATSAPP|LOCAL · pedidos.tipo_entrega ENVIO|RECOGIDA · metodos_pago.tipo MANUAL|PASARELA · resoluciones.tipo_documento FACTURA|NOTA_CREDITO.
- **CHECK valores**: producto_variantes.stock ≥ 0 (insumos.stock SÍ puede quedar negativo: el consumo es estimado y no debe frenar producción; genera alerta para ajuste) · cantidad > 0 (detalle_pedido, carrito_items, detalle_factura) · montos ≥ 0 · pagos.monto > 0 · reembolsos.monto > 0 · porcentaje_poliester 0–100 · rango_desde ≤ consecutivo_actual ≤ rango_hasta.
- detalle_movimiento: CHECK ((uid_variante IS NULL) <> (id_insumo IS NULL)).
- metodos_pago: CHECK (tipo = 'MANUAL' OR id_pasarela IS NOT NULL).
- pedidos: CHECK (tipo_entrega = 'RECOGIDA' OR direccion_envio IS NOT NULL).
- usuarios: CHECK (email IS NOT NULL OR telefono IS NOT NULL OR anonimizado_en IS NOT NULL).
- variables_historial: CHECK ((id_variable IS NULL) <> (id_ajuste IS NULL)).
- pagos: CHECK (moneda = 'COP') · CHECK destino IN (CAJA, BANCO, PASARELA).
- **ON DELETE**: RESTRICT por defecto. CASCADE solo en tablas hijas sin valor histórico: usuarios_roles, refresh_tokens, carrito_items, disenos_archivos, producto_imagenes, zonas_medidas. SET NULL en variables_historial.
- **Diseño ↔ variante**: UQ disenos(uid_diseno, uid_variante) + FK compuesta (uid_diseno, uid_variante) → disenos en detalle_pedido y en carrito_items (esta última ON UPDATE CASCADE, por si el cliente cambia la talla del borrador).
- detalle_pedido y carrito_items: CHECK (uid_diseno IS NULL OR id_tecnica IS NOT NULL).
- pasarelas: CHECK ambiente IN (SANDBOX, PRODUCCION).
- **UQ compuestos**: ajustes_precio_producto(uid_producto, clave) · zonas_impresion(uid_producto, codigo) · archivos_produccion(uid_detalle_pedido, id_zona) · pagos_eventos(id_pasarela, referencia_evento) · facturas(id_resolucion, numero) · notas_credito(id_resolucion, numero) · producto_imagenes(uid_producto, orden).
- **UQ NULLS NOT DISTINCT**: zonas_medidas(id_zona, id_talla) · parametros_tecnica(id_tecnica, uid_producto) · producto_variantes(uid_producto, id_talla, color_nombre, material).
- **UQ parciales**: direcciones(uid_usuario) WHERE predeterminada · datos_facturacion(uid_usuario) WHERE predeterminado · resoluciones_facturacion(tipo_documento) WHERE activa · facturas(uid_pedido) WHERE estado <> 'ANULADA'.


#### Reglas de negocio (backend, cada una en una sola transacción)

- **Diseños**: el carrito usa el diseño BORRADOR. Al crear el pedido se copia a uno BLOQUEADO (uid_diseno_origen = el borrador) y el detalle apunta a la copia. Un BLOQUEADO nunca se edita.
- **Máquina de estados**: PENDIENTE_PAGO → PAGADO → EN_PRODUCCION → LISTO → (ENVIO: ENVIADO, exige fila en envios) → ENTREGADO. RECOGIDA: LISTO → ENTREGADO. CANCELADO solo desde PENDIENTE_PAGO o PAGADO. Cada cambio escribe en pedidos_historial.
- **Totales**: pedidos.total = subtotal − descuento + impuestos + costo_envio. Precios congelados al crear el pedido (desglose_precio).
- **Pagos**: cobro al cliente = monto + recargo_cliente (recargo solo si metodos_pago.trasladar_comision). total_pagado = Σ monto APROBADO − Σ reembolsos COMPLETADO; se recalcula con bloqueo (@Version) al cambiar un pago o reembolso. PAGADO cuando total_pagado ≥ total (o % de anticipo configurable).
- **Webhooks**: validar firma → INSERT en pagos_eventos (UQ evita duplicados) → procesar → marcar procesado_en. Nunca confiar en la redirección del navegador.
- **Stock**: UPDATE … SET stock = stock − n WHERE id = ? AND stock ≥ n (0 filas = sin stock) + movimiento en la misma transacción. Se descuenta al CREAR el pedido (reserva). Si sigue PENDIENTE_PAGO cuando vence pedidos.expira_en, SIN pagos aprobados ni pagos PENDIENTE, un job lo cancela y devuelve el stock. Si una pasarela aprueba un pago de un pedido ya CANCELADO, se genera alerta y reembolso (la plata nunca se pierde en silencio). Al cancelar siempre se devuelve.
- **Consecutivos**: SELECT … FOR UPDATE sobre la resolución activa; validar rango y vigencia. Número de pedido con SEQUENCE (CL-2026-000123), nunca MAX()+1.
- **Facturas**: no se borran ni se editan; se anulan con nota crédito. Una factura válida por pedido.
- **Archivos**: el job de limpieza solo borra archivos sin referencias. Originales de diseños BLOQUEADOS nunca se borran. PNG de producción: se pueden borrar N días después de ENTREGADO (se regeneran).
- **Finanzas**: compras de inventario SOLO en movimientos (motivo COMPRA); gastos SOLO operativos (sin categoría de compras). costo_real = Σ (cantidad × costo_unitario) de los movimientos del pedido (VENTA, CONSUMO, INCIDENTE; DEVOLUCION resta) + Σ incidentes.perdida_adicional (pérdidas que no son inventario: mano de obra, reenvío) + envios.costo. Los materiales perdidos en un incidente van SOLO como movimiento INCIDENTE, para no contarlos dos veces. Comisiones de pasarela = Σ pagos.comision. Envíos y comisiones NO se registran como gastos.
  - **Utilidad del mes** = ingresos (Σ monto + recargo_cliente de pagos APROBADO − reembolsos) − Σ costo_real de pedidos − gastos − comisiones.
  - **Flujo de caja** = ingresos − compras (movimientos COMPRA) − gastos − envíos − comisiones.
- **Precios**: todo cambio en variables_globales o ajustes_precio_producto escribe en variables_historial. Los costos de técnica (DTF por metro, papel, tinta) viven SOLO en variables_globales; tecnicas.config guarda solo datos de proceso (ancho del rollo, margen).
- **Costo de insumos**: cada COMPRA recalcula insumos.costo_unitario con promedio ponderado; los movimientos guardan el costo del momento.
- **Producción para stock (TERMINADOS hechos en casa)**: un movimiento PRODUCCION con líneas negativas (variantes en blanco + insumos consumidos) y una línea positiva (la variante TERMINADA que entra). Costo unitario del terminado = Σ costo de lo que salió ÷ unidades producidas; actualiza producto_variantes.costo_compra con promedio ponderado. Todo en una sola transacción.
- **Reembolsos y notas**: Σ reembolsos de un pago ≤ pagos.monto; Σ notas crédito de una factura ≤ facturas.total. Si el pedido ya tenía factura, el reembolso genera nota crédito.
- **Validaciones cruzadas**: facturas usan una resolución FACTURA y notas una NOTA_CREDITO; archivos_produccion.id_zona debe ser del producto de la variante del detalle; el número de pedido reinicia por año.
- **Sesión**: refresh rotativo; si llega un token ya reemplazado → revocar toda su familia.
- **Habeas Data (Ley 1581)**: usuarios con pedidos no se borran, se anonimizan (anonimizado_en, activo = false) junto con sus direcciones y datos_facturacion. Las facturas conservan su snapshot porque la ley exige guardarlas.
- **Tipo de producto**: PERSONALIZABLE exige uid_diseno; TERMINADO lo exige NULL. Mismo diseño en otra talla = duplicar el diseño (cada talla tiene su zona y su PNG).
- **Descuentos**: el descuento del pedido se prorratea en las líneas de la factura.
- **Técnica**: validar que id_tecnica esté en producto_tecnicas del producto, y SUBLIMACION solo si porcentaje_poliester ≥ 70.
### 5.3 Datos semilla (`V2__seed.sql`)

- `roles`: ADMIN, EMPLEADO, CLIENTE.
- `tecnicas`: SUBLIMACION, DTF. `config` del DTF solo con datos de proceso: `{"anchoRolloCm":58,"largoHojaCm":100,"margenCm":1.0}`.
- `tallas`: XS, S, M, L, XL, XXL (con `orden`), más las de niño que se usen.
- `metodos_pago`: TRANSFERENCIA, NEQUI, EFECTIVO (MANUAL, activos); WOMPI_TARJETA, WOMPI_PSE, WOMPI_NEQUI (PASARELA, inactivos).
- `pasarelas`: WOMPI, ambiente SANDBOX, `activo = false`.
- `resoluciones_facturacion`: una FACTURA interna (prefijo FI, rango 1–999999) y una NOTA_CREDITO interna (prefijo NCI).
- `categorias_gasto`: Luz, Internet, Hosting/Dominio, Arriendo, Publicidad, Mantenimiento equipos, Otros. **No** existe la categoría de compras a proveedor (las compras son movimientos).
- `variables_globales` (fórmula v1, migrada del Excel; prueba de concepto que se va a reformular):
  - `valor_hora` = 8750
  - `factor_base` = 1.10 (significado pendiente de confirmar; ver `memory/decisions/`)
  - `porcentaje_imprevistos`, `porcentaje_administracion`, `porcentaje_utilidad` (valores generales; los del Excel por producto van en `ajustes_precio_producto`)
  - `envio_taller` (monto fijo por unidad)
  - `dtf_costo_metro` = 23000 (hoja de 58 × 100 cm = 5.800 cm² → $3,97/cm²)
  - `porcentaje_anticipo_minimo` (porcentaje del total que debe estar pagado para pasar a PAGADO; 100 = pago completo)
  - Opcionales en 0 al inicio: `costo_cm2_sublimacion`, `overhead_por_pedido`, `recargo_por_elemento_extra`.

### 5.4 Fórmula de precio (v1, editable)

```
costo  = costo_compra_variante + valor_hora × (min_diseño + min_fabricación) / 60 + envio_taller
         + impresión (DTF: área cobrada × costo por cm²; sublimación: área × costo_cm2_sublimacion)
precio = ROUNDDOWN_1000( costo × (factor_base + imprevistos + administración + utilidad) )
```

- Área cobrada DTF por diseño = (ancho + 2·margen) × (alto + 2·margen); margen por defecto 1 cm.
- Cada porcentaje se busca primero en `ajustes_precio_producto` del producto y, si no hay, en `variables_globales`.
- `precio_venta` de la variante, si existe, reemplaza la fórmula (precio fijo).
- Cada `detalle_pedido` guarda `precio_unitario` y `desglose_precio` (componente → valor). Cambiar variables nunca altera pedidos viejos.
- Endpoint de simulación: dado variante + diseño, devuelve precio y desglose con los valores actuales, o con valores propuestos sin guardarlos.
- Si se usa Wompi con `trasladar_comision`, el recargo al cliente se calcula con gross-up: `recargo = (neto + fijo) / (1 − pct) − neto`, y se guarda en `pagos.recargo_cliente`.

## 6. Flujos principales

### Cliente personaliza y compra

1. Entra al catálogo → elige producto → elige color y talla (ve la guía de tallas).
2. Abre el personalizador: sube imágenes o agrega texto por zona; mueve, escala, rota; alterna vista 2D/3D. Las zonas usan las medidas de `zonas_medidas` de su talla.
3. El sistema muestra el DPI efectivo y avisa si la imagen se ve pixelada.
4. Escribe un comentario opcional → "Agregar al carrito" (el diseño queda BORRADOR y la línea va a `carrito_items`).
5. Checkout: dirección o recogida, método de pago, datos de facturación → se crea el pedido PENDIENTE_PAGO en una transacción: copia BLOQUEADO de cada diseño, reserva de stock (movimiento VENTA), precio y desglose congelados, `expira_en`, historial, vaciado del carrito.
6. Pago manual: ve los datos de cuenta o Nequi y sube el comprobante (pago PENDIENTE). Puede abonar en varias partes.
7. Un empleado confirma → `total_pagado` se recalcula → PAGADO cuando cubre el anticipo mínimo.
8. Sigue el estado en "Mis pedidos" y ve la guía cuando se despacha.

### Pedido por WhatsApp o en el local

Un empleado busca o crea al cliente (puede ser sin acceso), arma el pedido desde el panel (`canal` WHATSAPP o LOCAL, `uid_creado_por` = el empleado) y registra el pago. Desde ahí sigue el mismo flujo.

### Producción

1. Llega la notificación "pedido nuevo". El empleado lo toma (`uid_asignado_a`) → EN_PRODUCCION.
2. Ve cada ítem con sus PNG por zona (generados al pasar a PAGADO), técnica, parámetros, talla, color, comentario y alertas de DPI.
3. Descarga e imprime. Si algo falla, registra un incidente (movimiento INCIDENTE con los materiales perdidos + `perdida_adicional` con lo que no es inventario).
4. Registra el consumo de insumos (movimiento CONSUMO) → LISTO. Se calcula `costo_real`.
5. Envío: crea la fila en `envios` → ENVIADO → ENTREGADO. Recogida: LISTO → ENTREGADO.

### Facturación

Al pasar a PAGADO se emite la factura interna (consecutivo con `FOR UPDATE`, `cliente_snapshot`, detalle con IVA, PDF). Una cancelación o devolución sobre un pedido facturado genera nota crédito (total o parcial).

### Producción para stock

Un empleado registra un movimiento PRODUCCION: salen variantes en blanco e insumos y entra la variante TERMINADA. El costo unitario del terminado se calcula y actualiza su `costo_compra`.

### Admin

Gestiona catálogo, variantes, tallas, zonas y medidas, parámetros de técnica, insumos, proveedores, compras, empleados, variables de precio, métodos de pago, resoluciones, gastos, y ve reportes.

## 7. Generación del PNG de producción

- Se dispara cuando el pedido pasa a PAGADO. Es un job asíncrono por línea y zona, con estado en `archivos_produccion` (PENDIENTE → GENERANDO → LISTO | ERROR) y reintentos (`intentos`, `error`).
- Fuente: el diseño **BLOQUEADO** de la línea, nunca el borrador.
- Por zona: lienzo de `ancho_cm / 2.54 × 300` por `alto_cm / 2.54 × 300` píxeles, transparente. Se pintan las capas en orden z con su posición, tamaño, rotación y espejo. Los textos usan las mismas fuentes que el editor (los .ttf viven en el backend y en el frontend).
- Se guarda en R2 con metadatos de 300 DPI (chunk pHYs). Nombre: `CL-2026-000123_item2_FRENTE_30x40cm.png`.
- Prueba obligatoria: un diseño conocido produce un PNG con dimensiones exactas y la capa en la posición esperada (tolerancia ±1 px).
- Opción "recortar al contenido" para DTF (ahorra film): genera además un PNG recortado al bounding box de las capas, con su posición anotada.
- Retención: el PNG puede borrarse N días después de ENTREGADO (`eliminar_despues_de`). Se regenera desde el diseño bloqueado y sus originales, que nunca se borran.

## 8. Archivos

- Bucket privado en Cloudflare R2. La base guarda `storage_key`, nunca una URL.
- Subida: el backend valida sesión, tipo y tamaño → crea `archivos` (SUBIENDO) → devuelve una URL firmada de subida → el navegador sube directo a R2 → el backend verifica el tipo real, dimensiones y sha256 → LISTO o RECHAZADO.
- Lectura: el backend valida permisos y genera una URL firmada de lectura de vida corta.
- Tipos permitidos: PNG, JPG, WEBP, SVG sanitizado, PDF; máximo 25 MB.
- Limpieza: un job borra solo archivos sin referencias cuyo `eliminar_despues_de` ya pasó. Nunca borra los originales de diseños BLOQUEADOS.
- Backups: el bucket también se respalda fuera de R2 (rclone).

## 9. UX y diseño visual

- **Tienda (cliente):** fondo oscuro (#0E0E10 aprox.), acentos con el degradado del logo (rojo #E53935 → naranja #FF8A00 → amarillo #FFD400 → verde #3DDC84), tipografía display tipo script para títulos de marca y una sans legible para el resto. Mobile-first, botones grandes, editor usable con el dedo (pinch para escalar, dos dedos para rotar).
- **Panel de producción/admin:** tema claro, denso y legible, tablas con filtros, pensado para trabajar horas.
- Logo en `assets/logo.png`.
- Accesibilidad básica: contraste AA, foco visible, textos alternativos.

## 10. Seguridad

- Contraseñas con BCrypt. Login rechazado si `password_hash` es NULL (cliente sin acceso).
- JWT de acceso corto (15 min) + refresh rotativo con familia: se guarda el hash del token; si llega un token ya reemplazado, se revoca toda su familia.
- Autorización por rol en cada endpoint (`/api/admin/**` → ADMIN/EMPLEADO según recurso).
- Un cliente solo ve sus pedidos, diseños, pagos, facturas y archivos.
- Archivos: bucket privado y URLs firmadas (§8). Validar tipo real y tamaño; recalcular dimensiones en el servidor.
- Webhooks de pasarela: validar firma, guardar el evento, procesar idempotente. El estado de un pago nunca se toma de la redirección del navegador.
- Rate limit en login, subida de archivos y creación de pagos.
- Secretos (JWT, R2, Wompi) solo por variables de entorno; nunca en la base ni en el repositorio.
- La aplicación se conecta a Postgres con un usuario de permisos mínimos (no superusuario).

## 11. Fases (ver prompts/)

- **Fundación (1 agente):** esqueleto, `common/`, `auth/`, almacenamiento de archivos (R2), OpenAPI inicial, infra inicial. El esquema SQL ya viene escrito (§5).
- **Ola 1 en paralelo:** catálogo + inventario · precios + finanzas + facturación · editor 2D · tienda cliente.
- **Ola 2 en paralelo:** pedidos + pagos + producción + envíos (backend) · panel admin/producción · visor 3D · render de PNG.
- **Ola 3:** integración + E2E · despliegue.

## 12. Propiedad de carpetas (para trabajar en paralelo sin pisarse)

| Agente | Es dueño de |
| --- | --- |
| A0 Fundación | todo el esqueleto, `common/` (incluye `AlmacenamientoArchivos` y UUID v7), `auth/`, `docs/openapi.yaml` (versión inicial), `infra/` inicial |
| A1 Catálogo e inventario | `backend/.../catalog/`, `backend/.../inventory/` |
| A2 Precios, finanzas y facturación | `backend/.../pricing/`, `backend/.../finance/`, `backend/.../billing/` (resoluciones, facturas, notas crédito, `InvoiceProvider`) |
| A3 Editor 2D | `frontend/src/features/editor2d/`, `frontend/src/shared/design/` (tipos y utilidades del JSON de diseño) |
| A4 Tienda cliente | `frontend/src/features/store/`, `features/account/`, `features/auth/` (UI) |
| B1 Pedidos, pagos y producción (backend) | `backend/.../orders/`, `payments/`, `production/`, `shipping/`, `notifications/` (llama a billing para emitir la factura) |
| B2 Panel admin | `frontend/src/features/admin/` |
| B3 Visor 3D | `frontend/src/features/viewer3d/`, `frontend/public/models/` |
| B4 Render de PNG | `backend/.../design/`, `docs/design-schema.json`, `backend/src/main/resources/fonts/` |
| C1 Integración y QA | `frontend/e2e/`, arreglos transversales coordinados |
| C2 Despliegue | `infra/` |

Archivos compartidos (`docs/openapi.yaml`, `frontend/src/shared/api/`, `frontend/src/app/router.tsx`, `application.yml`) se tocan siguiendo las reglas de CLAUDE.md §Coordinación.

**Migraciones Flyway:** `V1__schema.sql` y `V2__seed.sql` los escribe Andrés. Ningún agente modifica una migración ya aplicada. Si un agente necesita otra migración, la pide en `memory/requests/` y, una vez aprobada, usa su rango: A1 V100–V199, A2 V200–V299, B1 V300–V399, B4 V400–V499, C1 V500–V599. `spring.flyway.out-of-order=true` solo en dev.

## 13. Pruebas obligatorias

Cada regla de negocio de §5.2 tiene al menos un test de integración contra Postgres real (Testcontainers). Mínimo:

- Dos pedidos simultáneos por la última unidad → uno falla, el stock queda en 0.
- El mismo webhook dos veces → el pago se aprueba una sola vez.
- Doble clic en pagar (misma `clave_idempotencia`) → un solo pago.
- Dos facturas simultáneas → consecutivos distintos y seguidos; factura anulada → su número no se reutiliza.
- Pedido vencido sin pagos → CANCELADO y stock devuelto; pedido vencido con pago PENDIENTE → no se cancela.
- Pago aprobado por pasarela sobre pedido CANCELADO → alerta y reembolso.
- Diseño BLOQUEADO → cualquier edición falla.
- Línea con variante distinta a la del diseño → el INSERT falla (FK compuesta).
- Reembolsos que superan el monto del pago → rechazado.
- Transición de estado no permitida (PAGADO → ENTREGADO) → rechazada.
- Incidente → materiales en movimiento INCIDENTE y `costo_real` sin doble conteo.
- Producción para stock → costo del terminado = suma de lo consumido ÷ unidades.
- Utilidad y flujo de caja de un mes con datos conocidos → valores exactos.
- Job de limpieza → no borra archivos referenciados.
- Refresh token reutilizado → toda la familia revocada.
