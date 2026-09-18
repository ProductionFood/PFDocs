# HU-19 · Lotes de producto terminado — Especificación

**Fase 6** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** registrar lotes de producto terminado (código, producto, fechas, cantidad),
> **para que** sepa qué hay en estante.

**Depende de:** [HU-11](../HU-11/) (productos)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `codigo_lote` obligatorio y **único** | `UNIQUE KEY codigo_lote`; compartido con los lotes de materia prima (HU-09) |
| CA-02 | `id_producto` debe existir | Validado contra `ProductoRepository` |
| CA-03 | `fecha_produccion` y `fecha_vencimiento` obligatorias | `CHECK (vencimiento >= produccion)` en V2 §C-10 |
| CA-04 | `cantidad` obligatoria (decimal) | `CHECK (cantidad > 0)` en V2 |
| CA-05 | **Se crea automáticamente un registro** en `inventario_producto_terminado` | Transaccional; es lo que convierte el lote en stock vendible |
| CA-06 | Listar lotes por producto o por estado de vencimiento | `?idProducto=`, `?soloVencidos=`, `?proximosAVencer=` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/lotes-producto` | Listar con filtros y paginación | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/lotes-producto/{id}` | Obtener un lote con su saldo | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/lotes-producto` | Registrar un lote | ADMIN, PRODUCCION |
| `GET` | `/api/v1/lotes-producto/{id}/movimientos` | Kardex del lote | ADMIN, PRODUCCION, VENTAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · 🔑 Esta historia es la que crea el stock vendible

Sin lotes de producto terminado, `inventario_producto_terminado` está vacío y **HU-17 no
puede vender nada**: cualquier intento devuelve `STOCK_INSUFICIENTE`.

Por eso el `00-base/10-ROADMAP.md` la sitúa **antes** de HU-17, aunque su número sea mayor.
Planificar HU-17 antes que HU-19 hace que la historia no se pueda ni probar.

### R-02 · El alta es transaccional: lote + inventario (CA-05)

Al registrar un lote, en la misma transacción:

1. `INSERT` en `lotes` con `id_producto` informado e `id_materia_prima` a `NULL`;
2. `INSERT` en `inventario_producto_terminado` con `cantidad_disponible = cantidad`;
3. `INSERT` en `movimientos_inventario_pt` con tipo `ENTRADA_PRODUCCION`.

Las tres o ninguna. Un lote sin su fila de inventario es un registro que no sirve para nada:
no aparece en el stock, no se puede vender, y nada indica que falte algo.

La relación es `1:1` — `inventario_producto_terminado.id_lote` es `UNIQUE`.

### R-03 · El saldo vive en el inventario, no en el lote

`lotes.cantidad` es la cantidad con la que el lote **ingresó** y **nunca se modifica**.
`inventario_producto_terminado.cantidad_disponible` es lo que **queda**, y baja con cada
venta (HU-17) y sube con cada devolución (HU-18).

Confundirlos es el error natural al leer el esquema: `lotes.cantidad` parece el saldo pero
es el histórico de entrada. Consultar `lotes.cantidad` para saber qué hay disponible
devuelve siempre la cantidad original, como si nunca se hubiera vendido nada.

### R-04 · Un lote vencido deja de ser stock vendible
`v_stock_producto_terminado` filtra `fecha_vencimiento >= curdate()` (§C-02). El lote sigue
existiendo con su saldo, pero no cuenta como disponible para HU-17.

Es intencionado: el producto sigue físicamente en el estante hasta que alguien lo retira,
pero el sistema no permite venderlo.

### R-05 · No se registran lotes con vencimiento pasado
A diferencia de los lotes de materia prima (HU-09 R-05), aquí **sí se bloquea**: registrar
producto terminado ya vencido no tiene caso operativo — nacería sin poder venderse.
→ `400 LOTE_YA_VENCIDO`.

### R-06 · Un lote es de producto **o** de materia prima
`CHECK ck_lotes_exclusividad` de V2 §C-10. En esta historia `id_producto` va informado e
`id_materia_prima` va `NULL`. Al revés en HU-09.

### R-07 · Relación con los planes de producción
Lo natural sería que HU-21, al registrar la cantidad producida, generara el lote
automáticamente. **Las historias no lo piden y no se implementa**: el lote se registra aquí,
a mano.

Se documenta como mejora evidente para una fase posterior. Por ahora, producir y registrar
el lote son dos acciones separadas, y nada impide olvidar la segunda.

### R-08 · El código de lote se normaliza a mayúsculas
Igual que en HU-09 R-06. El `UNIQUE` es *case-insensitive* por la collation, así que
`pf-2026-001` y `PF-2026-001` colisionan de todos modos.

---

## 4. Modelo de datos

**Tabla `lotes`** (compartida con HU-09)

| Columna | Tipo | Notas |
|---|---|---|
| `id_lote` | `int unsigned` PK AI | |
| `codigo_lote` | `varchar(30)` NOT NULL **UNIQUE** | CA-01 |
| `id_producto` | `int unsigned` NULL | **Informado aquí** (R-06) |
| `id_materia_prima` | `int unsigned` NULL | `NULL` en esta historia |
| `fecha_produccion` | `date` NOT NULL | CA-03 |
| `fecha_vencimiento` | `date` NOT NULL | CA-03 · `CHECK >= produccion` |
| `cantidad` | `decimal(10,2)` NOT NULL | CA-04 · **cantidad de ingreso**, no saldo (R-03) |

**Tabla `inventario_producto_terminado`**

| Columna | Tipo | Notas |
|---|---|---|
| `id_inv_producto` | `int unsigned` PK AI | |
| `id_lote` | `int unsigned` NOT NULL **UNIQUE** FK | Relación 1:1 (R-02) |
| `cantidad_disponible` | `decimal(10,2)` NOT NULL | **Saldo actual** · `CHECK >= 0` (V2) |
| `fecha_ingreso_estante` | `datetime` NOT NULL DEFAULT `CURRENT_TIMESTAMP` | |

```json
// POST /api/v1/lotes-producto
{ "codigoLote": "PF-2026-0918", "idProducto": 3,
  "fechaProduccion": "2026-09-18", "fechaVencimiento": "2026-09-22",
  "cantidad": 25.00 }

// 201 Created
{ "idLote": 31, "codigoLote": "PF-2026-0918",
  "producto": { "idProducto": 3, "nombre": "Pan francés", "unidad": "und" },
  "fechaProduccion": "2026-09-18", "fechaVencimiento": "2026-09-22",
  "cantidadIngresada": 25.00, "cantidadDisponible": 25.00,
  "diasParaVencer": 4, "vencido": false,
  "stockTotalProducto": 102.00 }
```

Se devuelven **`cantidadIngresada` y `cantidadDisponible` por separado** (R-03): al crear
coinciden, pero después divergen y la distinción importa.

**Campos ordenables:** `fechaVencimiento`, `fechaProduccion`, `codigoLote`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `CODIGO_LOTE_DUPLICADO` | 409 | Ya existe un lote con ese código (incluidos los de materia prima) |
| `PRODUCTO_INEXISTENTE` | 409 | El `idProducto` no existe |
| `FECHAS_INVALIDAS` | 400 | `fechaVencimiento` anterior a `fechaProduccion` |
| `LOTE_YA_VENCIDO` | 400 | La fecha de vencimiento ya pasó (R-05) |
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

### Sobre la trazabilidad

Este lote es el que HU-17 consume por FEFO y el que HU-18 repone al aprobar una devolución.
La cadena completa —qué lote se produjo, a qué cliente fue y qué volvió— queda en
`movimientos_inventario_pt`.

Es la información que se necesita ante un retiro sanitario: saber qué clientes recibieron
producto de un lote determinado. Por eso el kardex de esta historia expone el historial
completo del lote, no solo su saldo.
