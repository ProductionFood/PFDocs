# HU-13 · Registro de compras — Especificación

**Fase 5** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** comprador,
> **quiero** registrar una compra a proveedor con fecha y estado,
> **para que** documente las adquisiciones de materia prima.

**Depende de:** [HU-06](../HU-06/) (proveedores)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `id_proveedor` debe existir | Validado contra `ProveedorRepository`; se exige además que esté **activo** |
| CA-02 | `fecha` obligatoria | `@NotNull`; por defecto la fecha actual |
| CA-03 | `estado` obligatorio | Valores canónicos: `PENDIENTE`, `RECIBIDA`, `CANCELADA` — `CHECK` en V2 §C-12 |
| CA-04 | Se retorna la compra creada con su `id_compra` | `201` + `Location` + `CompraResponse` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/compras` | Listar con filtros y paginación | ADMIN, COMPRAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/compras/{id}` | Obtener compra con su detalle | ADMIN, COMPRAS, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/compras` | Registrar una compra | ADMIN, COMPRAS |
| `PUT` | `/api/v1/compras/{id}` | Editar cabecera (solo si está PENDIENTE) | ADMIN, COMPRAS |
| `PATCH` | `/api/v1/compras/{id}/estado` | Cambiar estado · **la recepción suma stock** | ADMIN, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · 🔑 Recibir una compra es el punto principal de entrada de stock

Es la corrección de fondo que hizo falta al esquema (`02-CORRECCIONES-DB.md` §C-03): el
sistema descontaba materia prima pero **nada la sumaba**.

Al pasar una compra a `RECIBIDA`, en una única transacción:

1. Se valida que la compra tenga al menos una línea de detalle.
2. Por **cada línea** de `detalle_compra` se llama a
   `InventarioService.registrarEntrada(idMateriaPrima, cantidad, ENTRADA_COMPRA, "detalle_compra", idDetalle)`.
3. Se actualiza `compras.estado = 'RECIBIDA'`.
4. Se registra en bitácora con acción `RECEPCION_COMPRA`.

Si falla cualquier línea, **no entra ninguna**. Una recepción a medias deja un inventario
que nadie puede explicar.

### R-02 · `RECIBIDA` es un estado terminal

```
PENDIENTE ──▶ RECIBIDA      (irreversible)
    │
    └───────▶ CANCELADA
```

**No se permite volver de `RECIBIDA` a `PENDIENTE`.** Una vez que el stock entró, deshacerlo
con un cambio de estado obligaría a restar de un inventario que quizá ya se consumió en
producción, y el saldo podría quedar negativo.

Una recepción equivocada se corrige con un **movimiento de ajuste** explícito, que queda en
el kardex con su motivo. La corrección es visible; deshacer el estado sería invisible.

Cualquier transición no permitida devuelve `409 TRANSICION_INVALIDA`
(`04-CONTRATO-API.md` §7).

### R-03 · Una compra sin detalle no se puede recibir
Recibir una compra vacía no sumaría nada y dejaría el estado en `RECIBIDA` sin efecto,
imposibilitando después agregarle líneas (R-04). Se devuelve `409 COMPRA_SIN_DETALLE`.

### R-04 · El detalle solo se modifica mientras está `PENDIENTE`
Agregar, editar o quitar líneas (HU-14) de una compra ya recibida alteraría el inventario
sin dejar rastro del cambio. Una vez `RECIBIDA` o `CANCELADA`, el detalle queda congelado.

### R-05 · Cancelar una compra recibida no devuelve el stock
`RECIBIDA` es terminal (R-02), así que no puede cancelarse. Solo se cancela desde
`PENDIENTE`, cuando aún no ha entrado nada.

### R-06 · Se puede comprar a un proveedor y que luego se desactive
La validación de proveedor activo (CA-01) es **solo al crear**. Las compras ya registradas
de un proveedor desactivado siguen su curso normal, incluida su recepción: la mercancía ya
fue pedida.

### R-07 · La fecha puede ser pasada, no futura
Se registran compras de días anteriores. Una fecha futura carece de sentido para una
adquisición ya realizada → `400 FECHA_FUTURA`.

---

## 4. Modelo de datos

**Tabla `compras`** (V2 añade el `CHECK` de estado y el índice `idx_compras_fecha`)

| Columna | Tipo | Notas |
|---|---|---|
| `id_compra` | `int unsigned` PK AI | |
| `id_proveedor` | `int unsigned` NOT NULL FK | CA-01 |
| `fecha` | `date` NOT NULL | CA-02, R-07 |
| `estado` | `varchar(30)` NOT NULL | CA-03 · `CHECK IN ('PENDIENTE','RECIBIDA','CANCELADA')` |

```json
// POST /api/v1/compras
{ "idProveedor": 3, "fecha": "2026-09-18", "estado": "PENDIENTE" }

// 201 Created · Location: /api/v1/compras/12
{ "idCompra": 12,
  "proveedor": { "idProveedor": 3, "nombre": "Harinas del Caribe S.A.S." },
  "fecha": "2026-09-18", "estado": "PENDIENTE",
  "totalCompra": 0.00, "cantidadItems": 0, "editable": true }

// PATCH /api/v1/compras/12/estado
{ "estado": "RECIBIDA" }

// 200 OK — el stock ya entró
{ "idCompra": 12, "estado": "RECIBIDA", "totalCompra": 480000.00,
  "movimientosGenerados": [
    { "materiaPrima": "Harina de trigo", "cantidad": 100.00,
      "saldoAnterior": 12.50, "saldoPosterior": 112.50 },
    { "materiaPrima": "Azúcar", "cantidad": 50.00,
      "saldoAnterior": 45.00, "saldoPosterior": 95.00 }
  ]
}
```

Devolver los movimientos generados hace visible el efecto de la recepción en el momento
mismo, sin ir a otra pantalla a comprobarlo.

**Campos ordenables:** `fecha`, `estado`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `PROVEEDOR_INEXISTENTE` | 409 | El `idProveedor` no existe |
| `PROVEEDOR_INACTIVO` | 409 | El proveedor está desactivado (solo al crear) |
| `TRANSICION_INVALIDA` | 409 | Transición de estado no permitida (R-02) |
| `COMPRA_SIN_DETALLE` | 409 | Se intenta recibir una compra sin líneas (R-03) |
| `COMPRA_NO_EDITABLE` | 409 | Se intenta modificar una compra que no está `PENDIENTE` (R-04) |
| `FECHA_FUTURA` | 400 | La fecha de compra es posterior a hoy (R-07) |

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

### La recepción es la operación de mayor impacto del sistema

Es el punto donde el inventario crece, y por eso está restringida a `ADMIN` y `COMPRAS`
(`00-base/03-MATRIZ-ROLES.md`). Su contrapartida —el descuento por producción— está en
HU-23.

Entre las dos sostienen todo el control de inventario: si cualquiera de las dos falla, los
saldos dejan de significar algo y HU-10, HU-26 y HU-27 informan sobre datos falsos.
