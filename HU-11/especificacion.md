# HU-11 · Gestión de productos — Especificación

**Fase 4** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** CRUD de productos terminados (nombre, descripción, precio, unidad, estado),
> **para que** mantenga el catálogo de lo que vende la panadería.

**Depende de:** [HU-07](../HU-07/) (unidades de medida)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Nombre obligatorio, máx. 100 | `@NotBlank` + `@Size(max=100)` |
| CA-02 | Descripción opcional, máx. 150 | `@Size(max=150)` |
| CA-03 | Precio obligatorio (decimal positivo) | `@NotNull` + `@DecimalMin`; `CHECK (precio >= 0)` en V2 |
| CA-04 | `id_unidad` debe existir | Validado contra `UnidadMedidaRepository` |
| CA-05 | Estado por defecto activo (`1`) | `DEFAULT 1` **añadido en V2** §C-06 |
| CA-06 | Listar con paginación y filtrar por nombre | `GET /productos?nombre=&page=&size=` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/productos` | Listar con filtro y paginación | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/productos/{id}` | Obtener uno | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/productos` | Registrar | ADMIN |
| `PUT` | `/api/v1/productos/{id}` | Editar | ADMIN |
| `PATCH` | `/api/v1/productos/{id}/estado` | Activar o desactivar | ADMIN |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Cambiar el precio **no** afecta a los pedidos ya registrados

Gracias a `detalle_pedido.precio_unitario`, añadido en V2 (§C-01), el precio se copia en el
momento de agregar la línea al pedido (HU-17) y nunca se recalcula.

Sin esa columna, subir el precio del pan hoy cambiaría retroactivamente el valor de todos
los pedidos del último año, y los reportes darían un número distinto cada vez que alguien
edita el catálogo.

**Editar el precio aquí solo afecta a las ventas futuras.**

### R-02 · Un producto sin receta no se puede planificar
HU-12 asocia la receta; HU-21 y HU-22 la necesitan para calcular el consumo teórico. Un
producto sin receta puede venderse (si hay stock de lotes), pero **no puede entrar en un
plan de producción** con consumo teórico calculado.

El listado indica qué productos tienen receta, para que el hueco sea visible.

### R-03 · Desactivar un producto no retira su stock
HU-19 puede tener lotes de ese producto con existencias. Desactivar significa "no vender
más ni planificar producción nueva", no "desapareció del estante". HU-17 CA-02 exige
producto activo para agregarlo a un pedido.

### R-04 · La unidad del producto y la de la receta pueden diferir
`productos.id_unidad` es la unidad de **venta** (unidad, docena, bandeja).
`recetas.id_unidad` es la unidad de **producción** (HU-12 CA-02).

Un pan se vende por unidad pero la receta produce 30 unidades por tanda. Son dos conceptos
distintos y el esquema los separa correctamente.

### R-05 · No se elimina
`detalle_pedido`, `lotes`, `recetas` y `detalle_plan_produccion` lo referencian. Solo se
desactiva.

### R-06 · Precio cero permitido
`CHECK (precio >= 0)`, no `> 0`. Permite registrar productos promocionales o muestras
gratuitas. CA-03 dice "positivo", que se interpreta como "no negativo".

---

## 4. Modelo de datos

**Tabla `productos`**

| Columna | Tipo | Notas |
|---|---|---|
| `id_producto` | `int unsigned` PK AI | |
| `nombre` | `varchar(100)` NOT NULL | CA-01 |
| `descripcion` | `varchar(150)` NULL | CA-02 |
| `precio` | `decimal(10,2)` NOT NULL | CA-03 · `CHECK >= 0` (V2) |
| `id_unidad` | `int unsigned` NOT NULL FK | CA-04 · unidad de **venta** (R-04) |
| `estado` | `tinyint(1)` NOT NULL **DEFAULT 1** | CA-05 · corregido en V2 §C-06 |

```json
// POST /api/v1/productos
{ "nombre": "Pan francés", "descripcion": "Pan de corteza crujiente, 80 g",
  "precio": 1200.00, "idUnidad": 5 }

// 201 Created
{ "idProducto": 3, "nombre": "Pan francés",
  "descripcion": "Pan de corteza crujiente, 80 g", "precio": 1200.00,
  "unidad": { "idUnidad": 5, "nombre": "Unidad", "abreviatura": "und" },
  "activo": true, "tieneReceta": false, "stockDisponible": 0.00 }
```

`tieneReceta` y `stockDisponible` se calculan con `LEFT JOIN` a `recetas` y a
`v_stock_producto_terminado` (V2 §C-02). Son de solo lectura y hacen el listado útil sin
navegar a otras pantallas.

**Campos ordenables:** `nombre`, `precio`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `UNIDAD_INEXISTENTE` | 409 | El `idUnidad` no existe |
| `PRODUCTO_CON_STOCK` | 409 | Advertencia al desactivar un producto con existencias (no bloqueante) |

Los transversales (`VALIDACION_FALLIDA`, `NO_AUTENTICADO`, `SIN_PERMISO`,
`RECURSO_NO_ENCONTRADO`) están en `00-base/04-CONTRATO-API.md` §5.

---

## 6. Fuera de alcance

Lo que **no** cubre esta historia y no debe implementarse aquí:
se limita estrictamente a los criterios de aceptación listados arriba.
Cualquier funcionalidad adicional se propone como historia nueva, no se agrega en silencio.

---

## 7. Definición de Terminado

Se aplica la DoD de `00-base/01-CONVENCIONES.md` §7.

## 8. Notas

### Esta historia es gemela de HU-08

`productos` y `materias_primas` tienen casi la misma estructura y el mismo CRUD. Se
implementa copiando el patrón de HU-08, con dos diferencias:

- Productos tiene `descripcion`; materias primas no.
- **Productos NO crea fila de inventario al darse de alta.** El stock de producto terminado
  vive en `inventario_producto_terminado`, que es **por lote** (HU-19), no por producto.
  Es la asimetría del esquema que obligó a crear la vista `v_stock_producto_terminado`
  (§C-02).
