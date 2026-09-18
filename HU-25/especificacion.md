# HU-25 · Consulta de pedidos — Especificación

**Fase 8** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** consultar pedidos por cliente, rango de fechas o estado,
> **para que** dé seguimiento a las ventas.

**Depende de:** [HU-17](../HU-17/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Filtrar por `id_cliente` | `?idCliente=` |
| CA-02 | Filtrar por rango de fechas | `?fechaInicio=&fechaFin=` — ambos inclusivos |
| CA-03 | Filtrar por `estado` | `?estado=PENDIENTE|EN_PREPARACION|ENTREGADO|CANCELADO` |
| CA-04 | Cada pedido muestra su detalle (productos, cantidades) | `?incluirDetalle=true` o consulta individual |
| CA-05 | Soporta paginación | `page`, `size`, `sort` — `04-CONTRATO-API.md` §4 |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/pedidos` | Consulta con filtros combinables | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/pedidos/resumen` | Totales agregados del período | ADMIN, VENTAS |
| `GET` | `/api/v1/pedidos/por-cliente` | Ventas agrupadas por cliente | ADMIN, VENTAS |
| `GET` | `/api/v1/pedidos/productos-mas-vendidos` | Ranking de productos | ADMIN, VENTAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Solo lectura
Amplía los filtros del listado de HU-16 y agrega tres endpoints de agregación. El estado de
los pedidos se cambia en HU-16.

### R-02 · 🔑 Los importes históricos son fiables gracias a §C-01

`detalle_pedido.precio_unitario`, añadido en V2, es lo que permite que este reporte
signifique algo: **cada pedido se valora con el precio que tenía cuando se vendió**.

Sin esa columna, el total de un pedido de hace tres meses se recalcularía con el precio de
hoy, y el reporte de ventas de agosto daría un número distinto cada vez que alguien edita el
catálogo. Es la diferencia entre un informe y una cifra que cambia sola.

### R-03 · Los pedidos cancelados no cuentan como venta
En los totales agregados, `CANCELADO` se **excluye**: no se vendió nada.

`ENTREGADO` es venta consumada; `PENDIENTE` y `EN_PREPARACION` son venta comprometida. Se
informan **por separado**, igual que en HU-15 R-04: sumarlos todos daría una cifra que no
corresponde ni a lo facturado ni a lo prometido.

### R-04 · Las devoluciones aprobadas se descuentan del total
Un pedido de 30 panes con 3 devueltos y aprobados representa una venta neta de 27.

El reporte muestra **venta bruta, devoluciones y venta neta**. Informar solo la bruta
sobrevalora las ventas; informar solo la neta oculta el volumen de devoluciones, que es un
indicador de calidad.

### R-05 · Rango de fechas inclusivo
`fecha_pedido` es `DATE`, así que `BETWEEN` funciona sin sorpresas (mismo razonamiento que
HU-15 R-03). `fechaInicio > fechaFin` → `400 RANGO_FECHAS_INVALIDO`.

### R-06 · Cargar el detalle de todos los pedidos es costoso
Igual que en HU-15 R-06: con `incluirDetalle=true` el `size` se limita a **20** y la consulta
usa un único `JOIN` con agrupación en memoria, nunca una consulta por pedido.

### R-07 · El ranking de productos usa cantidades, no importes
"Más vendido" se mide por unidades despachadas. Medirlo por importe haría que un producto
caro vendido tres veces superara a uno barato vendido doscientas, lo cual responde a otra
pregunta.

Se exponen ambas columnas y se ordena por cantidad.

---

## 4. Modelo de datos

Sin cambios de esquema. V2 añadió `idx_pedidos_fecha (fecha_pedido, estado)`, que cubre
CA-02 y CA-03.

```json
// GET /api/v1/pedidos?idCliente=7&fechaInicio=2026-09-01&fechaFin=2026-09-18&estado=ENTREGADO
{
  "content": [
    { "idPedido": 45, "cliente": { "idCliente": 7, "nombre": "Tienda La Esquina" },
      "fechaPedido": "2026-09-16", "fechaEntrega": "2026-09-16",
      "estado": "ENTREGADO", "cantidadItems": 2,
      "totalBruto": 8400.00, "devoluciones": 0.00, "totalNeto": 8400.00 }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}

// GET /api/v1/pedidos/resumen?fechaInicio=2026-09-01&fechaFin=2026-09-30
{
  "periodo": { "desde": "2026-09-01", "hasta": "2026-09-30" },
  "entregados":   { "cantidad": 24, "total": 1840000.00 },
  "enCurso":      { "cantidad":  6, "total":  320000.00 },
  "cancelados":   { "cantidad":  2, "total":   45000.00 },
  "ventaBruta":      1840000.00,
  "devoluciones":      62000.00,
  "ventaNeta":       1778000.00,
  "ticketPromedio":    76666.67
}

// GET /api/v1/pedidos/productos-mas-vendidos?fechaInicio=2026-09-01&fechaFin=2026-09-30
[
  { "producto": "Pan francés", "unidadesVendidas": 1240.00,
    "importe": 1488000.00, "pedidos": 18 },
  { "producto": "Croissant", "unidadesVendidas": 320.00,
    "importe": 800000.00, "pedidos": 12 }
]
```

**Campos ordenables:** `fechaPedido`, `fechaEntrega`, `estado`, `total`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `RANGO_FECHAS_INVALIDO` | 400 | `fechaInicio` posterior a `fechaFin` |
| `CLIENTE_INEXISTENTE` | 404 | El `idCliente` del filtro no existe |

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

### Este reporte es el que valida la corrección §C-01

Si el total de un pedido cambia al editar el precio del producto en el catálogo, la columna
`precio_unitario` no se está usando y toda la información de ventas es inestable.

Es el caso de prueba CP-14 de esta historia, y el mismo que CP-16 de HU-11 verifica desde el
otro extremo.
