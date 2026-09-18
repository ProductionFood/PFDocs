# HU-14 · Detalle de compra — Especificación

**Fase 5** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** comprador,
> **quiero** agregar detalles a la compra (materia prima, cantidad, precio unitario),
> **para que** el sistema registre los ítems y calcule el total.

**Depende de:** [HU-13](../HU-13/), [HU-08](../HU-08/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `id_compra` debe existir | Validado; además debe estar `PENDIENTE` (HU-13 R-04) |
| CA-02 | `id_materia_prima` debe existir | Validado contra `MateriaPrimaRepository` |
| CA-03 | `cantidad` y `precio_unitario` obligatorios (decimales) | `@NotNull`; `CHECK (cantidad > 0)` y `CHECK (precio >= 0)` en V2 |
| CA-04 | Calcular el total de la compra | `SUM(cantidad * precio_unitario)` — calculado por el backend, nunca por el cliente |
| CA-05 | Listar el detalle de una compra | `GET /compras/{id}/detalles` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/compras/{idCompra}/detalles` | Listar el detalle | ADMIN, COMPRAS, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/compras/{idCompra}/detalles` | Agregar un ítem | ADMIN, COMPRAS |
| `PUT` | `/api/v1/compras/{idCompra}/detalles/{id}` | Editar cantidad o precio | ADMIN, COMPRAS |
| `DELETE` | `/api/v1/compras/{idCompra}/detalles/{id}` | Quitar un ítem | ADMIN, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · El detalle solo se modifica mientras la compra está `PENDIENTE`

Es la regla de HU-13 R-04 aplicada aquí. Modificar una línea de una compra ya `RECIBIDA`
alteraría el total sin que el inventario correspondiente cambie: el documento diría una
cosa y el stock reflejaría otra, sin rastro del desajuste.

Los cuatro endpoints verifican el estado de la compra antes de actuar →
`409 COMPRA_NO_EDITABLE`.

### R-02 · Una materia prima no puede repetirse en la misma compra

V2 añadió `UNIQUE (id_compra, id_materia_prima)` (§C-13). Sin esa restricción, dos líneas de
harina en la misma compra harían que la recepción de HU-13 sumara dos veces por caminos
distintos y que el total fuera ambiguo de interpretar.

Al agregar una materia prima ya presente → `409 MATERIA_PRIMA_DUPLICADA`.
**Lo correcto es editar la línea existente**, no crear otra.

### R-03 · El total lo calcula el backend, siempre

```
subtotal_línea = cantidad × precio_unitario
total_compra   = Σ subtotales
```

Con `BigDecimal` y `RoundingMode.HALF_UP`. **El frontend nunca calcula el total que se
guarda o se informa**; puede mostrar una previsualización mientras se escribe, pero la cifra
oficial viene del servidor.

Si ambas capas calcularan, dos reglas de redondeo distintas producirían diferencias de
centavos imposibles de explicar (`04-CONTRATO-API.md` §6).

### R-04 · El precio de la compra es independiente del costo de referencia
`detalle_compra.precio_unitario` es lo que **realmente se pagó** en esta compra concreta.
`materias_primas.costo_unitario` (HU-08 R-03) es un valor de referencia para costear recetas.

**Registrar una compra no actualiza el costo de referencia**: hacerlo lo sobrescribiría con
el precio puntual de la última compra, que puede ser atípico por volumen o negociación.

La interfaz muestra ambos para que la diferencia sea visible y quien compra pueda juzgarla.

### R-05 · El precio puede ser cero
`CHECK (precio_unitario >= 0)`, no `> 0`: hay muestras y bonificaciones. La cantidad, en
cambio, debe ser estrictamente positiva (`CHECK > 0`): una línea de cero unidades no es
una línea.

### R-06 · Se permite eliminar líneas (mientras esté `PENDIENTE`)
Igual que en HU-12 R-07: una línea de detalle no tiene valor histórico propio mientras el
documento no se haya ejecutado. Una vez `RECIBIDA`, nada del detalle se toca.

### R-07 · Las cantidades del detalle son `decimal(10,2)`
A diferencia de `detalle_receta`, que usa 3 decimales (HU-12 R-06). Se compra en kilos y
litros enteros o con dos decimales; no hace falta más precisión.

---

## 4. Modelo de datos

**Tabla `detalle_compra`** (V2 añade el `UNIQUE` y dos `CHECK`)

| Columna | Tipo | Notas |
|---|---|---|
| `id_detalle_compra` | `int unsigned` PK AI | |
| `id_compra` | `int unsigned` NOT NULL FK | CA-01 |
| `id_materia_prima` | `int unsigned` NOT NULL FK | CA-02 |
| `cantidad` | `decimal(10,2)` NOT NULL | CA-03 · `CHECK > 0` · R-07 |
| `precio_unitario` | `decimal(10,2)` NOT NULL | CA-03 · `CHECK >= 0` · R-05 |

**`UNIQUE KEY uk_detalle_compra (id_compra, id_materia_prima)`** — añadido en V2 §C-13.

```json
// POST /api/v1/compras/12/detalles
{ "idMateriaPrima": 5, "cantidad": 100.00, "precioUnitario": 3200.00 }

// 201 Created
{ "idDetalleCompra": 28,
  "materiaPrima": { "idMateriaPrima": 5, "nombre": "Harina de trigo", "unidad": "kg" },
  "cantidad": 100.00, "precioUnitario": 3200.00, "subtotal": 320000.00,
  "costoReferencia": 3200.00, "diferenciaCosto": 0.00,
  "totalCompra": 320000.00 }

// GET /api/v1/compras/12/detalles
{
  "idCompra": 12, "estado": "PENDIENTE", "editable": true,
  "detalles": [
    { "idDetalleCompra": 28, "materiaPrima": {...}, "cantidad": 100.00,
      "precioUnitario": 3200.00, "subtotal": 320000.00 },
    { "idDetalleCompra": 29, "materiaPrima": {...}, "cantidad": 50.00,
      "precioUnitario": 3200.00, "subtotal": 160000.00 }
  ],
  "totalCompra": 480000.00, "cantidadItems": 2
}
```

`diferenciaCosto` compara el precio pagado con el costo de referencia (R-04): hace visible
si esta compra salió más cara o más barata de lo previsto.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `COMPRA_INEXISTENTE` | 404 | La compra no existe |
| `COMPRA_NO_EDITABLE` | 409 | La compra no está `PENDIENTE` (R-01) |
| `MATERIA_PRIMA_INEXISTENTE` | 409 | El `idMateriaPrima` no existe o está inactiva |
| `MATERIA_PRIMA_DUPLICADA` | 409 | Esa materia prima ya está en la compra (R-02) |
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

### Este detalle es lo que la recepción convierte en inventario

Cada línea de `detalle_compra` genera un movimiento `ENTRADA_COMPRA` cuando HU-13 recibe la
compra. Por eso el `UNIQUE` de R-02 importa: una línea duplicada se traduciría directamente
en stock duplicado.

El mismo patrón de cabecera-detalle aparece en HU-12 (recetas), HU-17 (pedidos) y HU-21
(planes de producción). Las restricciones de unicidad de V2 §C-13 cubren los cuatro casos.
