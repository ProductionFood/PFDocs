# HU-10 · Consulta de inventario de materia prima — Especificación

**Fase 3** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** consultar el inventario de materia prima (cantidad disponible, última actualización),
> **para que** sepa qué tenemos en bodega.

**Depende de:** [HU-08](../HU-08/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Mostrar cada materia prima con su `cantidad_disponible` | Vista `v_inventario_materia_prima` (V2) |
| CA-02 | Mostrar la `fecha_actualizacion` de cada registro | `DEFAULT CURRENT_TIMESTAMP ON UPDATE` añadido en V2 §C-07 |
| CA-03 | Filtrar por materia prima | `?idMateriaPrima=` |
| CA-04 | Filtrar materia prima con stock **por debajo del mínimo** | `?soloStockBajo=true` — usa `estado_stock` de la vista |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/inventario` | Consultar inventario con filtros | ADMIN, PRODUCCION, COMPRAS, CONSULTA |
| `GET` | `/api/v1/inventario/{idMateriaPrima}` | Detalle de una materia prima | ADMIN, PRODUCCION, COMPRAS, CONSULTA |
| `GET` | `/api/v1/inventario/{idMateriaPrima}/movimientos` | Kardex: historial de movimientos | ADMIN, PRODUCCION, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Solo lectura
Esta historia **no modifica nada**. El inventario se mueve desde HU-09 (lotes), HU-13
(recepción de compras) y HU-23 (consumo de producción), siempre a través de
`InventarioService`. Aquí solo se consulta.

### R-02 · Se consulta la vista, no la tabla
`v_inventario_materia_prima` (creada en V2) resuelve en una sola consulta lo que si no
requeriría varios `JOIN` y un `CASE` repetido en cada endpoint:

- `LEFT JOIN` con `inventario` y `COALESCE(...,0)`: una materia prima sin fila aparece con
  cero en lugar de desaparecer del listado;
- `estado_stock` calculado: `AGOTADO` · `BAJO` · `OK`;
- `faltante`: cuánto falta para alcanzar el mínimo.

El `LEFT JOIN` es una red de seguridad, **no sustituye a HU-08 R-01**: sin fila real de
inventario no hay dónde sumar cuando llegue la primera compra.

### R-03 · El kardex es lo que da sentido al saldo
`GET /inventario/{id}/movimientos` devuelve el historial de
`movimientos_inventario_mp` (V2 §C-04): qué entró, qué salió, cuándo, quién y por qué
documento.

Sin esto, cuando el saldo no cuadre con la bodega no habrá forma de saber qué pasó. La
bitácora de HU-04 no sirve para eso: registra "alguien tocó la tabla inventario", no un
libro de movimientos consultable.

### R-04 · Se muestran también las materias primas inactivas con stock
Una materia prima desactivada (HU-08 R-04) conserva sus existencias. Ocultarla del
inventario haría que ese stock desapareciera del sistema aunque siga en la bodega.
Se muestra marcada como inactiva; el filtro por estado permite excluirla.

### R-05 · `estado_stock` y su significado

| Valor | Condición | Interpretación |
|---|---|---|
| `AGOTADO` | `disponible = 0` | No hay nada. Bloquea producción |
| `BAJO` | `0 < disponible < stock_minimo` | Hay que reabastecer (alimenta HU-27) |
| `OK` | `disponible >= stock_minimo` | Normal |

Con `stock_minimo = 0` (HU-08 R-06), nunca se marca `BAJO`: es la forma de decir "no
alertar por este insumo".

---

## 4. Modelo de datos

**Vista `v_inventario_materia_prima`** (creada en V2)

| Campo | Origen |
|---|---|
| `id_materia_prima`, `nombre` | `materias_primas` |
| `unidad` | `unidades_medida.abreviatura` |
| `stock_minimo`, `costo_unitario`, `estado` | `materias_primas` |
| `cantidad_disponible` | `COALESCE(inventario.cantidad_disponible, 0)` |
| `fecha_actualizacion` | `inventario` |
| `estado_stock` | Calculado: `AGOTADO` / `BAJO` / `OK` |
| `faltante` | `GREATEST(stock_minimo - disponible, 0)` |

```json
// GET /api/v1/inventario?soloStockBajo=true
{
  "content": [
    { "idMateriaPrima": 7, "nombre": "Levadura", "unidad": "g",
      "cantidadDisponible": 0.00, "stockMinimo": 500.00, "faltante": 500.00,
      "estadoStock": "AGOTADO", "activo": true,
      "fechaActualizacion": "2026-09-17T16:40:02" },
    { "idMateriaPrima": 5, "nombre": "Harina de trigo", "unidad": "kg",
      "cantidadDisponible": 12.50, "stockMinimo": 50.00, "faltante": 37.50,
      "estadoStock": "BAJO", "activo": true,
      "fechaActualizacion": "2026-09-18T09:12:44" }
  ],
  "page": 0, "size": 20, "totalElements": 2, "totalPages": 1
}

// GET /api/v1/inventario/5/movimientos
{
  "content": [
    { "fecha": "2026-09-18T09:12:44", "tipo": "SALIDA_PRODUCCION",
      "cantidad": 100.000, "saldoAnterior": 112.50, "saldoPosterior": 12.50,
      "origen": { "tabla": "consumo_materia_prima", "id": 31 },
      "usuario": "Ismael Batalla" },
    { "fecha": "2026-09-12T08:05:11", "tipo": "ENTRADA_LOTE",
      "cantidad": 100.000, "saldoAnterior": 12.50, "saldoPosterior": 112.50,
      "origen": { "tabla": "lotes", "id": 14 },
      "usuario": "Carlos Cochero" }
  ]
}
```

**Campos ordenables:** `nombre`, `cantidadDisponible`, `faltante`, `fechaActualizacion`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|


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

### `fecha_actualizacion` se mantiene sola

El esquema original declaraba `datetime NOT NULL` **sin valor por defecto**, aunque CA-02
pida mostrarla y HU-23 CA-02 exija actualizarla.

V2 (§C-07) la dejó como `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`: se
actualiza sola en cada `UPDATE`, sin que ningún servicio tenga que acordarse. Una fecha que
depende de que el programador la asigne en cada ruta de código acaba desactualizada en la
ruta que alguien olvidó.
