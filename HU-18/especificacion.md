# HU-18 · Registro de devoluciones — Especificación

**Fase 6** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** vendedor,
> **quiero** registrar devoluciones de un pedido con motivo, descripción y cantidad,
> **para que** controle las devoluciones de producto.

**Depende de:** [HU-17](../HU-17/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `id_detalle_pedido` debe existir | Validado contra `DetallePedidoRepository` |
| CA-02 | `motivo` obligatorio, máx. 50 | `@NotBlank` + `@Size(max=50)` |
| CA-03 | `descripcion` opcional, máx. 255 | `@Size(max=255)` |
| CA-04 | `cantidad_devuelta` obligatoria, **no puede exceder la cantidad del pedido** | Validado contra el acumulado de devoluciones previas |
| CA-05 | `fecha_devolucion` se registra automáticamente | `DEFAULT (curdate())` **añadido en V2** §C-07 |
| CA-06 | `estado` obligatorio | Canónicos: `PENDIENTE`, `APROBADA`, `RECHAZADA` — `CHECK` en V2 §C-12 |
| CA-07 | **Al aprobarse, la cantidad vuelve al inventario** de producto terminado | `InventarioPtService.registrarEntrada()` con tipo `ENTRADA_DEVOLUCION` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/devoluciones` | Listar con filtros y paginación | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/devoluciones/{id}` | Obtener una devolución | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/pedidos/{id}/devoluciones` | Devoluciones de un pedido | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/devoluciones` | Registrar una devolución | ADMIN, VENTAS |
| `PATCH` | `/api/v1/devoluciones/{id}/estado` | Aprobar o rechazar · **aprobar repone stock** | ADMIN, VENTAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · 🔑 El stock vuelve al **aprobar**, no al registrar

Una devolución nace `PENDIENTE`: alguien la reporta, pero la mercancía todavía no se ha
revisado. Reponer el stock en ese momento haría que producto en mal estado —que es el motivo
habitual de una devolución— volviera a estar disponible para vender.

Al pasar a `APROBADA`, y solo entonces, se repone. Transaccional.

### R-02 · ⚠️ A qué lote vuelve la mercancía

La devolución apunta a `detalle_pedido`, pero una línea de pedido pudo consumir **varios
lotes** (HU-17 R-02). La historia no dice a cuál devolver.

**Decisión:** se consulta el kardex (`movimientos_inventario_pt` con
`tabla_origen = 'detalle_pedido'`) para saber de qué lotes salió esa línea, y se devuelve
**en orden inverso al consumo** — al último lote del que se tomó primero.

Razón: el último lote consumido es el de vencimiento más lejano, y es el que más
probabilidades tiene de seguir siendo vendible. Devolver al lote más antiguo repondría
mercancía que quizá ya venció, inflando un stock que no se puede vender.

Si un lote de destino ya venció, la devolución se aprueba igualmente pero **la unidad no
vuelve como stock vendible**: `v_stock_producto_terminado` excluye vencidos (HU-17 R-03).
Queda registrada en el kardex para la trazabilidad.

### R-03 · No se puede devolver más de lo pedido (CA-04)

La validación es contra el **acumulado**, no contra cada devolución por separado:

```
Σ(devoluciones APROBADA + PENDIENTE de esa línea) + nueva  ≤  detalle_pedido.cantidad
```

Sin contar las `PENDIENTE`, se podrían registrar tres devoluciones de 10 sobre una línea de
10, y al aprobarlas todas volverían 30 unidades al inventario de una venta de 10.

Las `RECHAZADA` **no** cuentan: liberan la cantidad para una devolución posterior.

### R-04 · Máquina de estados

```
PENDIENTE ──▶ APROBADA      (repone stock · terminal)
    │
    └───────▶ RECHAZADA     (terminal)
```

Ambos son terminales. Una devolución aprobada por error se corrige con un ajuste de
inventario, no revirtiendo el estado — mismo criterio que la recepción de compra
(HU-13 R-02).

### R-05 · Solo se devuelve de pedidos entregados o en preparación
Devolver de un pedido `PENDIENTE` no tiene sentido: la mercancía no ha salido.
De uno `CANCELADO` tampoco: el stock ya volvió entero (HU-16 R-04), y aprobar una devolución
encima lo duplicaría.

### R-06 · La fecha la pone el servidor
`DEFAULT (curdate())`, añadido en V2 §C-07: el esquema original la exigía `NOT NULL` sin
valor por defecto, aunque CA-05 pidiera que fuera automática. Mismo cuidado con la zona
horaria que en HU-16 R-01.

### R-07 · El motivo es texto libre, con sugerencias
`varchar(50)` sin restricción. La interfaz ofrece motivos frecuentes —*Producto en mal
estado*, *Error en el pedido*, *Cliente insatisfecho*, *Producto vencido*— pero permite
escribir otro.

Normalizar el motivo en un catálogo sería mejor para reportes, pero no está en los criterios
de aceptación y el esquema no lo contempla. Se registra como mejora posible.

---

## 4. Modelo de datos

**Tabla `devoluciones`** (V2 añade el `DEFAULT` de fecha, el `CHECK` de estado y un índice)

| Columna | Tipo | Notas |
|---|---|---|
| `id_devolucion` | `int unsigned` PK AI | |
| `id_detalle_pedido` | `int unsigned` NOT NULL FK | CA-01 |
| `motivo` | `varchar(50)` NOT NULL | CA-02, R-07 |
| `descripcion` | `varchar(255)` NULL | CA-03 |
| `cantidad_devuelta` | `decimal(10,2)` NOT NULL | CA-04 · `CHECK > 0` |
| `fecha_devolucion` | `date` NOT NULL **DEFAULT (curdate())** | CA-05 · V2 §C-07 |
| `estado` | `varchar(30)` NOT NULL | CA-06 · `CHECK` en V2 §C-12 |

```json
// POST /api/v1/devoluciones
{ "idDetallePedido": 88, "motivo": "Producto en mal estado",
  "descripcion": "Tres panes llegaron aplastados en el transporte",
  "cantidadDevuelta": 3.00 }

// 201 Created
{ "idDevolucion": 15,
  "pedido": { "idPedido": 47, "cliente": "Tienda La Esquina" },
  "producto": { "idProducto": 3, "nombre": "Pan francés" },
  "cantidadPedida": 30.00, "cantidadDevuelta": 3.00,
  "devueltoPreviamente": 0.00, "disponibleParaDevolver": 27.00,
  "motivo": "Producto en mal estado",
  "fechaDevolucion": "2026-09-18", "estado": "PENDIENTE" }

// PATCH /api/v1/devoluciones/15/estado
{ "estado": "APROBADA" }

// 200 OK — el stock volvió
{ "idDevolucion": 15, "estado": "APROBADA",
  "lotesRepuestos": [
    { "codigoLote": "PF-2026-0917", "cantidad": 3.00,
      "saldoAnterior": 22.00, "saldoPosterior": 25.00 }
  ]
}

// 422 CANTIDAD_EXCEDE_PEDIDO
{ "code": "CANTIDAD_EXCEDE_PEDIDO",
  "message": "Se pidieron 30 unidades y ya hay 28 devueltas o en trámite. Solo puede devolver 2." }
```

**Campos ordenables:** `fechaDevolucion`, `estado`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `DETALLE_PEDIDO_INEXISTENTE` | 409 | El `idDetallePedido` no existe |
| `CANTIDAD_EXCEDE_PEDIDO` | 422 | La devolución supera lo pedido, contando las previas (R-03) |
| `PEDIDO_NO_DEVOLVIBLE` | 409 | El pedido está `PENDIENTE` o `CANCELADO` (R-05) |
| `TRANSICION_INVALIDA` | 409 | La devolución ya fue aprobada o rechazada (R-04) |
| `CANTIDAD_INVALIDA` | 400 | `cantidadDevuelta` menor o igual a cero |

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

### Por qué el `422` y no un `409`

`CANTIDAD_EXCEDE_PEDIDO` usa `422 Unprocessable Entity`: los datos son sintácticamente
correctos —un número positivo en un campo de cantidad— pero violan una regla de negocio que
depende del estado acumulado del sistema.

Es la distinción de `00-base/04-CONTRATO-API.md` §3: `409` para conflictos con algo que ya
existe (un duplicado, un estado incompatible), `422` para reglas de negocio que los datos
no satisfacen.
