# HU-15 · Historial de compras — Especificación

**Fase 5** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** comprador,
> **quiero** consultar el historial de compras por proveedor o rango de fechas,
> **para que** analice los gastos.

**Depende de:** [HU-14](../HU-14/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Filtrar por `id_proveedor` | `?idProveedor=` |
| CA-02 | Filtrar por rango de fechas | `?fechaInicio=&fechaFin=` — ambos inclusivos |
| CA-03 | Filtrar por `estado` | `?estado=PENDIENTE|RECIBIDA|CANCELADA` |
| CA-04 | Cada compra muestra sus detalles (materias primas, cantidades, subtotal) | `?incluirDetalle=true` o consulta individual |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/compras` | Historial con filtros combinables | ADMIN, COMPRAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/compras/resumen` | Totales agregados del período | ADMIN, COMPRAS |
| `GET` | `/api/v1/compras/por-proveedor` | Gasto agrupado por proveedor | ADMIN, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Historia de solo lectura
No modifica nada. Amplía los filtros del listado de HU-13 y agrega dos endpoints de
agregación. El estado de las compras se cambia en HU-13.

### R-02 · Los filtros se combinan con `AND`
Proveedor, rango de fechas y estado se aplican simultáneamente. Todos opcionales: sin
ninguno, devuelve el historial completo paginado.

### R-03 · Rango de fechas inclusivo en ambos extremos
`fechaInicio = 2026-09-01` y `fechaFin = 2026-09-18` incluyen las compras de ambos días.

Como `compras.fecha` es `DATE` (sin hora), basta `fecha BETWEEN :inicio AND :fin`. Si la
columna fuera `DATETIME`, una compra de las 15:00 del día final quedaría fuera al compararse
contra la medianoche — el error clásico de los filtros por fecha.

`fechaInicio > fechaFin` → `400 RANGO_FECHAS_INVALIDO`.

### R-04 · Las compras canceladas no cuentan como gasto
En los totales agregados, `CANCELADA` se **excluye**: nunca se pagó.

Las `PENDIENTE` se informan **aparte** de las `RECIBIDA`: son gasto comprometido pero no
ejecutado. Sumarlas todas en una sola cifra daría un número que no corresponde ni a lo
pagado ni a lo comprometido.

### R-05 · El total se calcula sobre el detalle, siempre
No se guarda ningún total en `compras`: se recalcula con
`SUM(cantidad * precio_unitario)` sobre `detalle_compra` en cada consulta.

Guardar un total denormalizado obligaría a mantenerlo sincronizado en cada alta, edición y
borrado de línea — y bastaría un camino olvidado para que el documento mintiera. Con el
volumen de una pyme, recalcular es instantáneo.

### R-06 · Cargar el detalle de todas las compras es costoso
`?incluirDetalle=true` sobre un historial de un año dispara el problema N+1 o produce una
respuesta enorme. Por eso:

- por defecto el listado **no** incluye detalles, solo el total y el número de ítems;
- con `incluirDetalle=true`, el `size` máximo se limita a **20**;
- la consulta usa un único `JOIN` con agrupación en memoria, no una consulta por compra.

---

## 4. Modelo de datos

Sin cambios de esquema. Consulta `compras`, `detalle_compra`, `proveedores` y
`materias_primas`. V2 añadió `idx_compras_fecha (fecha, estado)`, que cubre CA-02 y CA-03.

```json
// GET /api/v1/compras?idProveedor=3&fechaInicio=2026-09-01&fechaFin=2026-09-18&estado=RECIBIDA
{
  "content": [
    { "idCompra": 11, "proveedor": { "idProveedor": 3, "nombre": "Harinas del Caribe" },
      "fecha": "2026-09-15", "estado": "RECIBIDA",
      "cantidadItems": 4, "totalCompra": 1240000.00 }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}

// GET /api/v1/compras/resumen?fechaInicio=2026-09-01&fechaFin=2026-09-30
{
  "periodo": { "desde": "2026-09-01", "hasta": "2026-09-30" },
  "recibidas":  { "cantidad": 8, "total": 4850000.00 },
  "pendientes": { "cantidad": 2, "total":  620000.00 },
  "canceladas": { "cantidad": 1, "total":   85000.00 },
  "gastoEjecutado":  4850000.00,
  "gastoComprometido": 620000.00
}

// GET /api/v1/compras/por-proveedor?fechaInicio=2026-09-01&fechaFin=2026-09-30
[
  { "proveedor": "Harinas del Caribe S.A.S.", "cantidadCompras": 4,
    "total": 2340000.00, "porcentaje": 48.25 },
  { "proveedor": "Distribuidora Lácteos", "cantidadCompras": 3,
    "total": 1810000.00, "porcentaje": 37.32 }
]
```

`gastoEjecutado` y `gastoComprometido` van separados por R-04.

**Campos ordenables:** `fecha`, `estado`, `total`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `RANGO_FECHAS_INVALIDO` | 400 | `fechaInicio` posterior a `fechaFin` |
| `PROVEEDOR_INEXISTENTE` | 404 | El `idProveedor` del filtro no existe |

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

### Qué queda fuera

No se implementa exportación a Excel ni PDF: no está en los criterios de aceptación. Si el
equipo lo considera necesario, es una historia nueva.

Tampoco se comparan períodos entre sí (este mes contra el anterior). El endpoint de resumen
acepta cualquier rango, así que la comparación puede hacerse con dos llamadas desde el
cliente.
