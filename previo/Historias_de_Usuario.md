# ProductionFood — Historias de Usuario

Sistema Web para la Planificación y Gestión de la Producción en Pymes de Alimentos.

## Stack Tecnológico

| Capa | Tecnología | Versión |
|------|------------|---------|
| Lenguaje | Java | 25 (LTS) |
| Framework | Spring Boot | 4.1.x |
| Seguridad | Spring Security 7 + JWT (jjwt 0.13.0) | — |
| Acceso a datos | Spring JdbcTemplate (sin ORM) | — |
| Base de datos | MySQL en AWS RDS | 8.4 (LTS) |
| Driver | mysql-connector-j | 9.x |
| Build | Maven | 3.9.x |
| Documentación API | SpringDoc OpenAPI (Swagger UI) | 3.1.1 |
| IDE | IntelliJ IDEA Community | 2026.x |
| Validación | Jakarta Validation (spring-boot-starter-validation) | — |

> **Versiones revisadas en septiembre de 2026.** El stack declarado originalmente
> (Java 21, Spring Boot 3.3, Angular 17, SpringDoc 2.6.0) quedó fuera de soporte.
> **Amazon RDS para MySQL 8.0 terminó su soporte estándar el 31 de julio de 2026** y pasa
> a facturarse como Extended Support: se usa **MySQL 8.4 LTS**. Ver
> `docs/00-base/00-STACK-VERSIONES.md` para el detalle y la justificación.

### Decisiones técnicas
- **Sin ORM**: Se usa JdbcTemplate con RowMapper manual para control total del SQL.
- **JWT**: Autenticación stateless. Token en header `Authorization: Bearer <token>`.
- **Swagger UI**: Disponible en `/swagger-ui.html` para probar endpoints.
- **Contraseñas**: Almacenadas con hash (BCrypt via Spring Security).
- **DB**: MySQL 8.4 en AWS RDS (`jdbc:mysql://<endpoint-rds>:3306/productionfood`).
  En desarrollo local se usa MySQL 8.4, la misma versión, para evitar diferencias de motor.
- **Correcciones de esquema**: el `db.sql` original tiene huecos frente a estos criterios de aceptación. Ver `docs/00-base/02-CORRECCIONES-DB.md` y las migraciones versionadas.

---

## Stack Frontend

| Capa | Tecnología | Versión |
|------|------------|---------|
| Framework | Angular | 22.x |
| Lenguaje | TypeScript | 5.9.x |
| UI | Angular Material | 22.x |
| HTTP Client | Angular HttpClient | — |
| Routing | Angular Router | — |
| Formularios | Reactive Forms | — |
| Runtime | Node.js | 22 LTS / 24 LTS |
| Build | Angular CLI | 22.x |

### Decisiones técnicas frontend
- **Componentes**: standalone (por defecto desde Angular 19), con Angular Material para UI consistente.
- **Plantillas**: nueva sintaxis de control de flujo (`@if`, `@for`, `@switch`).
- **Estado**: signals para estado local del componente; sin NgRx en este alcance.
- **Comunicación HTTP**: Servicios con `HttpClient` contra la API REST (`/api/*`).
- **Autenticación**: JWT almacenado en `localStorage`, interceptor para adjuntar token.
- **Formularios**: Reactive Forms con validación del lado del cliente.
- **Estructura**: Módulos por dominio (productos, pedidos, compras, producción, reportes).

---

## Fase 1: Infraestructura y Seguridad

### HU-01: Registro de usuarios
**Como** administrador,
**quiero** registrar usuarios del sistema con nombre, correo, contraseña y rol,
**para que** controle quién tiene acceso al sistema.

**Criterios de aceptación:**
- CA-01: El nombre es obligatorio (máx. 100 caracteres).
- CA-02: El correo es obligatorio y debe ser único en el sistema.
- CA-03: La contraseña es obligatoria (máx. 200 caracteres, almacenada como hash).
- CA-04: El rol es obligatorio y debe existir en la tabla `roles`.
- CA-05: El estado por defecto es activo (`1`).
- CA-06: Se retorna el usuario creado con su `id_usuario`.

---

### HU-02: Gestión de usuarios
**Como** administrador,
**quiero** listar, editar y desactivar usuarios,
**para que** gestione quién tiene acceso al sistema.

**Criterios de aceptación:**
- CA-01: Se puede listar todos los usuarios con paginación.
- CA-02: Se puede filtrar por nombre o correo.
- CA-03: Se puede editar nombre, correo, rol y estado de un usuario existente.
- CA-04: Se puede desactivar/activar un usuario cambiando su `estado`.
- CA-05: No se permite eliminar un usuario (solo desactivar).

---

### HU-03: Inicio de sesión
**Como** usuario,
**quiero** iniciar sesión con correo y contraseña,
**para que** acceda a las funcionalidades según mi rol.

**Criterios de aceptación:**
- CA-01: Se recibe correo y contraseña.
- CA-02: Si las credenciales son correctas, se retorna un token JWT.
- CA-03: Si el usuario está desactivado, se retorna error 403.
- CA-04: Si las credenciales son incorrectas, se retorna error 401.
- CA-05: El token contiene el `id_usuario`, `correo` y `rol`.

---

### HU-04: Bitácora de auditoría
**Como** sistema,
**quiero** registrar en bitápora cada acción crítica (crear, editar, eliminar),
**para que** exista trazabilidad de las operaciones.

**Criterios de aceptación:**
- CA-01: Se registra automáticamente al crear, editar o eliminar registros.
- CA-02: Se almacena: `id_usuario`, `accion`, `tabla_afectada`, `detalle`, `fecha`.
- CA-03: La fecha se registra con `current_timestamp()` por defecto.
- CA-04: La bitápora es de solo lectura (no se edita ni elimina).

---

## Fase 2: Gestión de Datos Maestros

### HU-05: Gestión de clientes
**Como** administrador,
**quiero** crear, consultar, editar y desactivar clientes (nombre, contacto, teléfono, estado),
**para que** lleve el registro de quiénes compran.

**Criterios de aceptación:**
- CA-01: Nombre es obligatorio (máx. 100 caracteres).
- CA-02: Contacto y teléfono son opcionales.
- CA-03: Estado por defecto es activo (`1`).
- CA-04: Se puede listar con paginación y filtrar por nombre.
- CA-05: No se permite eliminar, solo desactivar.

---

### HU-06: Gestión de proveedores
**Como** administrador,
**quiero** CRUD de proveedores (nombre, contacto, teléfono, correo, estado),
**para que** gestione quiénes nos suministran materia prima.

**Criterios de aceptación:**
- CA-01: Nombre es obligatorio (máx. 100 caracteres).
- CA-02: Contacto, teléfono y correo son opcionales.
- CA-03: Estado por defecto es activo (`1`).
- CA-04: Se puede listar con paginación y filtrar por nombre.
- CA-05: No se permite eliminar, solo desactivar.

---

### HU-07: Unidades de medida
**Como** administrador,
**quiero** CRUD de unidades de medida (nombre, abreviatura),
**para que** estandarice las cantidades en todo el sistema.

**Criterios de aceptación:**
- CA-01: Nombre y abreviatura son obligatorios.
- CA-02: La abreviatura debe ser única.
- CA-03: Se puede listar todas las unidades disponibles.

---

## Fase 3: Inventario de Materias Primas

### HU-08: Gestión de materias primas
**Como** administrador,
**quiero** CRUD de materias primas (nombre, unidad, stock mínimo, costo unitario, estado),
**para que** conozca los ingredientes disponibles.

**Criterios de aceptación:**
- CA-01: Nombre es obligatorio (máx. 100 caracteres).
- CA-02: `id_unidad` debe existir en `unidades_medida`.
- CA-03: `stock_minimo` y `costo_unitario` son obligatorios (decimales).
- CA-04: Estado por defecto es activo (`1`).
- CA-05: Se puede listar con paginación y filtrar por nombre.

---

### HU-09: Registro de lotes de materia prima
**Como** administrador,
**quiero** registrar lotes de materia prima con código, fechas de producción/vencimiento y cantidad,
**para que** controle la vida útil de los ingredientes.

**Criterios de aceptación:**
- CA-01: `codigo_lote` es obligatorio y único (máx. 30 caracteres).
- CA-02: `id_materia_prima` debe existir en `materias_primas`.
- CA-03: `fecha_produccion` y `fecha_vencimiento` son obligatorias.
- CA-04: `cantidad` es obligatoria (decimal).
- CA-05: Se puede listar lotes por materia prima.

---

### HU-10: Consulta de inventario de materia prima
**Como** administrador,
**quiero** consultar el inventario de materia prima (cantidad disponible, última actualización),
**para que** sepa qué tenemos en bodega.

**Criterios de aceptación:**
- CA-01: Se muestra cada materia prima con su `cantidad_disponible`.
- CA-02: Se muestra la `fecha_actualizacion` de cada registro.
- CA-03: Se puede filtrar por materia prima.
- CA-04: Se puede filtrar materia prima con stock por debajo del mínimo.

---

## Fase 4: Catálogo de Productos y Recetas

### HU-11: Gestión de productos
**Como** administrador,
**quiero** CRUD de productos terminados (nombre, descripción, precio, unidad, estado),
**para que** mantenga el catálogo de lo que vende la panadería.

**Criterios de aceptación:**
- CA-01: Nombre es obligatorio (máx. 100 caracteres).
- CA-02: Descripción es opcional (máx. 150 caracteres).
- CA-03: Precio es obligatorio (decimal positivo).
- CA-04: `id_unidad` debe existir en `unidades_medida`.
- CA-05: Estado por defecto es activo (`1`).
- CA-06: Se puede listar con paginación y filtrar por nombre.

---

### HU-12: Gestión de recetas
**Como** administrador,
**quiero** asociar una receta a cada producto indicando materias primas y cantidades,
**para que** sepa qué ingredientes se necesitan por unidad.

**Criterios de aceptación:**
- CA-01: Cada producto puede tener una sola receta (`UNIQUE` en `id_producto`).
- CA-02: La receta tiene `nombre`, `cantidad_producir`, `id_unidad` y `estado`.
- CA-03: El detalle de receta lista materias primas con sus cantidades.
- CA-04: Se puede agregar, editar y eliminar materias primas del detalle.
- CA-05: No se permite duplicar la misma materia prima en una receta.

---

## Fase 5: Compras

### HU-13: Registro de compras
**Como** comprador,
**quiero** registrar una compra a proveedor con fecha y estado,
**para que** documente las adquisiciones de materia prima.

**Criterios de aceptación:**
- CA-01: `id_proveedor` debe existir en `proveedores`.
- CA-02: `fecha` es obligatoria.
- CA-03: `estado` es obligatorio (ej: "Pendiente", "Recibida", "Cancelada").
- CA-04: Se retorna la compra creada con su `id_compra`.

---

### HU-14: Detalle de compra
**Como** comprador,
**quiero** agregar detalles a la compra (materia prima, cantidad, precio unitario),
**para que** el sistema registre los ítems y calcule el total.

**Criterios de aceptación:**
- CA-01: `id_compra` debe existir en `compras`.
- CA-02: `id_materia_prima` debe existir en `materias_primas`.
- CA-03: `cantidad` y `precio_unitario` son obligatorios (decimales).
- CA-04: Se puede calcular el total de la compra (suma de `cantidad * precio_unitario`).
- CA-05: Se puede listar el detalle de una compra.

---

### HU-15: Historial de compras
**Como** comprador,
**quiero** consultar el historial de compras por proveedor o rango de fechas,
**para que** analice los gastos.

**Criterios de aceptación:**
- CA-01: Se puede filtrar por `id_proveedor`.
- CA-02: Se puede filtrar por rango de fechas (`fecha_inicio`, `fecha_fin`).
- CA-03: Se puede filtrar por `estado`.
- CA-04: Cada compra muestra sus detalles (materias primas, cantidades, subtotal).

---

## Fase 6: Pedidos y Ventas

### HU-16: Creación de pedidos
**Como** vendedor,
**quiero** crear un pedido para un cliente con fecha de entrega y estado,
**para que** gestione las ventas.

**Criterios de aceptación:**
- CA-01: `id_cliente` debe existir en `clientes` y estar activo.
- CA-02: `fecha_pedido` se registra automáticamente con la fecha actual.
- CA-03: `fecha_entrega` es opcional.
- CA-04: `estado` es obligatorio (ej: "Pendiente", "En preparación", "Entregado", "Cancelado").
- CA-05: Se retorna el pedido creado con su `id_pedido`.

---

### HU-17: Agregar productos al pedido
**Como** vendedor,
**quiero** agregar productos al pedido con cantidad,
**para que** se registren los ítems y se descuente del inventario de producto terminado.

**Criterios de aceptación:**
- CA-01: `id_pedido` debe existir en `pedidos`.
- CA-02: `id_producto` debe existir en `productos` y estar activo.
- CA-03: `cantidad` es obligatoria (decimal positivo).
- CA-04: Se verifica que haya stock suficiente en `inventario_producto_terminado`.
- CA-05: Se puede listar el detalle de un pedido.

---

### HU-18: Registro de devoluciones
**Como** vendedor,
**quiero** registrar devoluciones de un pedido con motivo, descripción y cantidad,
**para que** controle las devoluciones de producto.

**Criterios de aceptación:**
- CA-01: `id_detalle_pedido` debe existir en `detalle_pedido`.
- CA-02: `motivo` es obligatorio (máx. 50 caracteres).
- CA-03: `descripcion` es opcional (máx. 255 caracteres).
- CA-04: `cantidad_devuelta` es obligatoria (no puede exceder la cantidad original del pedido).
- CA-05: `fecha_devolucion` se registra automáticamente.
- CA-06: `estado` es obligatorio (ej: "Pendiente", "Aprobada", "Rechazada").
- CA-07: Al aprobarse, se devuelve la cantidad al inventario de producto terminado.

---

### HU-19: Lotes de producto terminado
**Como** administrador,
**quiero** registrar lotes de producto terminado (código, producto, fechas, cantidad),
**para que** sepa qué hay en estante.

**Criterios de aceptación:**
- CA-01: `codigo_lote` es obligatorio y único.
- CA-02: `id_producto` debe existir en `productos`.
- CA-03: `fecha_produccion` y `fecha_vencimiento` son obligatorias.
- CA-04: `cantidad` es obligatoria (decimal).
- CA-05: Se crea automáticamente un registro en `inventario_producto_terminado`.
- CA-06: Se puede listar lotes por producto o por estado de vencimiento.

---

## Fase 7: Producción

### HU-20: Creación de plan de producción
**Como** productor,
**quiero** crear un plan de producción con fechas, estado, observaciones y usuario responsable,
**para que** organice la fabricación.

**Criterios de aceptación:**
- CA-01: `fecha_planificacion` y `fecha_produccion` son obligatorias.
- CA-02: `estado` es obligatorio (ej: "Planificado", "En proceso", "Completado", "Cancelado").
- CA-03: `observaciones` es opcional (máx. 255 caracteres).
- CA-04: `id_usuario` debe existir en `usuarios`.
- CA-05: Se retorna el plan creado con su `id_plan`.

---

### HU-21: Productos en plan de producción
**Como** productor,
**quiero** agregar productos al plan de producción con cantidad planificada vs. producida,
**para que** controle la ejecución.

**Criterios de aceptación:**
- CA-01: `id_plan` debe existir en `planes_produccion`.
- CA-02: `id_producto` debe existir en `productos`.
- CA-03: `cantidad_planificada` es obligatoria (decimal positivo).
- CA-04: `cantidad_producida` inicia en `0.00` y se actualiza durante la producción.
- CA-05: Se puede listar el detalle de un plan de producción.

---

### HU-22: Registro de consumo de materia prima
**Como** productor,
**quiero** registrar el consumo real de materia prima por cada producto del plan,
**para que** compare con lo teórico y mida eficiencia.

**Criterios de aceptación:**
- CA-01: `id_detalle_plan` debe existir en `detalle_plan_produccion`.
- CA-02: `id_materia_prima` debe existir en `materias_primas`.
- CA-03: `cantidad_teorica` es obligatoria (viene de la receta).
- CA-04: `cantidad_real` es obligatoria (lo que realmente se usó).
- CA-05: `fecha_registro` se registra automáticamente.
- CA-06: Se puede calcular la eficiencia: `(cantidad_teorica / cantidad_real) * 100`.

---

### HU-23: Actualización automática de inventario por producción
**Como** administrador,
**quiero** que al registrar consumo de materia prima se actualice automáticamente el inventario,
**para que** los datos de stock sean consistentes.

**Criterios de aceptación:**
- CA-01: Al registrar un `consumo_materia_prima`, se descuenta `cantidad_real` del `inventario`.
- CA-02: Se actualiza la `fecha_actualizacion` en `inventario`.
- CA-03: Si el stock queda por debajo del `stock_minimo`, se genera una alerta (respuesta con flag).
- CA-04: La operación es transaccional (si falla, se revierte todo).

---

## Fase 8: Reportes y Consultas

### HU-24: Dashboard resumen
**Como** administrador,
**quiero** un dashboard con resumen del día (pedidos del día, compras pendientes, stock bajo, producción del día),
**para que** tenga visión general de la operación.

**Criterios de aceptación:**
- CA-01: Retorna el total de pedidos del día actual.
- CA-02: Retorna compras con estado "Pendiente".
- CA-03: Retorna materias primas con stock por debajo del mínimo.
- CA-04: Retorna planes de producción del día actual.
- CA-05: Retorna productos próximos a vencer (próximos 7 días).

---

### HU-25: Consulta de pedidos
**Como** administrador,
**quiero** consultar pedidos por cliente, rango de fechas o estado,
**para que** dé seguimiento a las ventas.

**Criterios de aceptación:**
- CA-01: Se puede filtrar por `id_cliente`.
- CA-02: Se puede filtrar por rango de fechas (`fecha_inicio`, `fecha_fin`).
- CA-03: Se puede filtrar por `estado`.
- CA-04: Cada pedido muestra su detalle (productos, cantidades).
- CA-05: Soporta paginación.

---

### HU-26: Eficiencia de producción
**Como** administrador,
**quiero** ver la eficiencia de producción (planificado vs. producido, teórico vs. real),
**para que** evalúe el rendimiento de la panadería.

**Criterios de aceptación:**
- CA-01: Muestra cada plan con sus productos: `cantidad_planificada` vs. `cantidad_producida`.
- CA-02: Muestra el consumo teórico vs. real por materia prima.
- CA-03: Calcula el porcentaje de eficiencia general.
- CA-04: Se puede filtrar por rango de fechas de producción.

---

### HU-27: Alerta de stock bajo
**Como** administrador,
**quiero** ver materias primas con stock por debajo del mínimo,
**para que** reabastezca a tiempo.

**Criterios de aceptación:**
- CA-01: Lista todas las materias primas donde `cantidad_disponible < stock_minimo`.
- CA-02: Muestra el nombre de la materia prima, stock actual y stock mínimo.
- CA-03: Muestra la diferencia (cuánto falta para alcanzar el mínimo).
- CA-04: Se puede ordenar por urgencia (mayor diferencia primero).
