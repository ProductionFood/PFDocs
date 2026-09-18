# HU-17 · Agregar productos al pedido — Especificación

**Fase 6** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** vendedor,
> **quiero** agregar productos al pedido con cantidad,
> **para que** se registren los ítems y se descuente del inventario de producto terminado.

**Depende de:** [HU-16](../HU-16/), **[HU-19](../HU-19/) (sin lotes de producto terminado no hay stock que vender)**

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `id_pedido` debe existir | Validado; además debe estar `PENDIENTE` o `EN_PREPARACION` |
| CA-02 | `id_producto` debe existir **y estar activo** | Validado contra `ProductoRepository.existeActivo()` |
| CA-03 | `cantidad` obligatoria (decimal positivo) | `@NotNull` + `CHECK (cantidad > 0)` en V2 |
| CA-04 | **Verificar stock suficiente** en `inventario_producto_terminado` | Vista `v_stock_producto_terminado` (V2 §C-02) + descuento FEFO |
| CA-05 | Listar el detalle de un pedido | `GET /pedidos/{id}/detalles` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/pedidos/{idPedido}/detalles` | Listar el detalle | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/productos/{id}/stock` | Stock disponible de un producto | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/pedidos/{idPedido}/detalles` | Agregar un producto | ADMIN, VENTAS |
| `PUT` | `/api/v1/pedidos/{idPedido}/detalles/{id}` | Cambiar la cantidad | ADMIN, VENTAS |
| `DELETE` | `/api/v1/pedidos/{idPedido}/detalles/{id}` | Quitar un producto | ADMIN, VENTAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · 🔑 El esquema no permitía cumplir CA-04 tal como estaba escrito

CA-04 pide verificar el stock en `inventario_producto_terminado`. Pero esa tabla está
indexada **por lote** (`id_lote` UNIQUE), no por producto:

```
inventario_producto_terminado(id_inv_producto, id_lote, cantidad_disponible, ...)
lotes(id_lote, codigo_lote, id_producto, ...)
```

Saber cuántos panes hay disponibles requiere agregar todos los lotes de ese producto. Y peor:
**la historia no decía de qué lote descontar** cuando hay varios.

Resuelto en V2 (`02-CORRECCIONES-DB.md` §C-02) con dos vistas:
`v_stock_producto_terminado` (agregado por producto, excluyendo lotes vencidos) y
`v_lotes_disponibles_fefo` (lotes ordenables por vencimiento).

### R-02 · 🔑 Política FEFO — *First Expired, First Out*

**Se descuenta primero del lote que vence antes.** Es la política correcta para alimentos
perecederos: descontar del lote más nuevo dejaría el viejo caducando en el estante.

```
Pedido: 30 panes
Lotes disponibles:
  PF-2026-0915  vence 19/09   12 und   ← se toman los 12
  PF-2026-0917  vence 21/09   40 und   ← se toman 18
  PF-2026-0918  vence 22/09   25 und   ← no se toca
```

Una línea de pedido puede consumir **varios lotes**. Cada lote consumido genera su propio
movimiento `SALIDA_VENTA` en `movimientos_inventario_pt`.

⚠️ **La vista no puede ordenar por sí sola**: MySQL ignora el `ORDER BY` declarado dentro de
una vista. El orden FEFO se aplica **en la consulta que la usa**, y está documentado en
`V2__correcciones.sql`. Una implementación que confíe en el `ORDER BY` de la vista dejará de
respetar FEFO sin dar ningún error.

### R-03 · Los lotes vencidos no son stock vendible
`v_stock_producto_terminado` filtra `fecha_vencimiento >= curdate()`. Un producto con 100
unidades vencidas tiene stock **cero** a efectos de venta.

### R-04 · El precio se copia al agregar la línea

Gracias a `detalle_pedido.precio_unitario`, añadido en V2 (§C-01), el precio se toma de
`productos.precio` **en el momento de agregar** y no se recalcula nunca.

Sin esa columna, subir el precio del pan hoy cambiaría el valor de todos los pedidos
históricos, y HU-25 daría un número distinto cada vez que alguien edita el catálogo.

### R-05 · Concurrencia: el último pan

Dos vendedores agregan simultáneamente el último pan disponible. Sin protección, ambos leen
`cantidad_disponible = 1`, ambos validan y ambos descuentan: el stock queda en `-1`.

Tres defensas, en orden:
1. `SELECT ... FOR UPDATE` sobre las filas de `inventario_producto_terminado` implicadas;
2. la validación de stock y el descuento **en la misma transacción**;
3. `CHECK (cantidad_disponible >= 0)` de V2 como última red.

Si el `CHECK` llega a dispararse, hay un fallo en 1 o 2.

### R-06 · Un producto no puede repetirse en el mismo pedido
`UNIQUE (id_pedido, id_producto)` añadido en V2 §C-13. Agregar uno ya presente →
`409 PRODUCTO_DUPLICADO`; lo correcto es **editar la cantidad** de la línea existente.

### R-07 · Editar la cantidad ajusta el stock por diferencia
- Aumentar de 10 a 15 → se descuentan **5** más (validando stock).
- Reducir de 10 a 6 → se **devuelven 4** a los lotes de origen.

No se devuelve todo y se vuelve a descontar: eso generaría movimientos de kardex que no
corresponden a ningún hecho real.

### R-08 · Quitar una línea devuelve todo el stock
`DELETE` devuelve la cantidad completa a los lotes de los que salió, con movimientos
`ENTRADA_DEVOLUCION`.

### R-09 · El detalle solo se modifica si el pedido no está cerrado
`PENDIENTE` y `EN_PREPARACION` admiten cambios. `ENTREGADO` y `CANCELADO`, no: la mercancía
ya salió o ya volvió. Una devolución posterior a la entrega es HU-18.

### R-10 · Trazabilidad lote ↔ línea de pedido
Como una línea puede consumir varios lotes, la correspondencia queda registrada **en el
kardex** (`movimientos_inventario_pt` con `tabla_origen = 'detalle_pedido'`), no en
`detalle_pedido`. Es la única forma de saber después qué lote fue a qué cliente — dato
necesario ante un retiro sanitario de producto.

---

## 4. Modelo de datos

**Tabla `detalle_pedido`** (V2 añade `precio_unitario`, el `UNIQUE` y el `CHECK`)

| Columna | Tipo | Notas |
|---|---|---|
| `id_detalle_pedido` | `int unsigned` PK AI | |
| `id_pedido` | `int unsigned` NOT NULL FK | CA-01 |
| `id_producto` | `int unsigned` NOT NULL FK | CA-02 |
| `cantidad` | `decimal(10,2)` NOT NULL | CA-03 · `CHECK > 0` |
| `precio_unitario` | `decimal(10,2)` NOT NULL | **Añadido en V2 §C-01** · R-04 |

**`UNIQUE KEY uk_detalle_pedido (id_pedido, id_producto)`** — V2 §C-13.

```json
// GET /api/v1/productos/3/stock
{ "idProducto": 3, "nombre": "Pan francés", "cantidadDisponible": 77.00,
  "lotesActivos": 3, "proximoVencimiento": "2026-09-19",
  "lotes": [
    { "codigoLote": "PF-2026-0915", "vence": "2026-09-19", "disponible": 12.00 },
    { "codigoLote": "PF-2026-0917", "vence": "2026-09-21", "disponible": 40.00 },
    { "codigoLote": "PF-2026-0918", "vence": "2026-09-22", "disponible": 25.00 }
  ]
}

// POST /api/v1/pedidos/47/detalles
{ "idProducto": 3, "cantidad": 30.00 }

// 201 Created
{ "idDetallePedido": 88,
  "producto": { "idProducto": 3, "nombre": "Pan francés", "unidad": "und" },
  "cantidad": 30.00, "precioUnitario": 1200.00, "subtotal": 36000.00,
  "lotesConsumidos": [
    { "codigoLote": "PF-2026-0915", "cantidad": 12.00, "vence": "2026-09-19" },
    { "codigoLote": "PF-2026-0917", "cantidad": 18.00, "vence": "2026-09-21" }
  ],
  "totalPedido": 36000.00 }

// 409 STOCK_INSUFICIENTE
{ "code": "STOCK_INSUFICIENTE",
  "message": "Solo hay 77 unidades disponibles de Pan francés. Se solicitaron 100." }
```

`lotesConsumidos` hace visible la aplicación de FEFO y sirve de comprobante de trazabilidad.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `PEDIDO_INEXISTENTE` | 404 | El pedido no existe |
| `PEDIDO_NO_EDITABLE` | 409 | El pedido está `ENTREGADO` o `CANCELADO` (R-09) |
| `PRODUCTO_INEXISTENTE` | 409 | El producto no existe |
| `PRODUCTO_INACTIVO` | 409 | El producto está desactivado |
| `PRODUCTO_DUPLICADO` | 409 | Ese producto ya está en el pedido (R-06) |
| `STOCK_INSUFICIENTE` | 409 | No hay unidades disponibles suficientes (CA-04) |
| `CANTIDAD_INVALIDA` | 400 | `cantidad` menor o igual a cero |

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

### Esta es la historia más compleja de la Fase 6

Combina cuatro cosas que pueden fallar de forma silenciosa: agregación de stock por lotes
(R-01), política FEFO (R-02), concurrencia (R-05) y ajuste por diferencia al editar (R-07).

Ninguna produce un error visible cuando se implementa mal:
- sin FEFO, el producto viejo caduca en el estante mientras se vende el nuevo;
- sin bloqueo, el stock queda negativo bajo concurrencia;
- sin ajuste por diferencia, el kardex registra movimientos que nunca ocurrieron.

Por eso `tarea-qa.md` dedica casos específicos a cada una.
