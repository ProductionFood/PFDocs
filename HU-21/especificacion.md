# HU-21 · Productos en plan de producción — Especificación

**Fase 7** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** productor,
> **quiero** agregar productos al plan de producción con cantidad planificada vs. producida,
> **para que** controle la ejecución.

**Depende de:** [HU-20](../HU-20/), [HU-11](../HU-11/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `id_plan` debe existir | Validado; además el plan no puede estar cerrado |
| CA-02 | `id_producto` debe existir | Validado contra `ProductoRepository` |
| CA-03 | `cantidad_planificada` obligatoria (decimal positivo) | `CHECK > 0` en V2 |
| CA-04 | `cantidad_producida` inicia en `0.00` y se actualiza durante la producción | `DEFAULT 0.00` ya en el esquema; endpoint propio para actualizarla |
| CA-05 | Listar el detalle de un plan | `GET /planes-produccion/{id}/detalles` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/planes-produccion/{idPlan}/detalles` | Listar los productos del plan | ADMIN, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/planes-produccion/{idPlan}/detalles` | Agregar un producto | ADMIN, PRODUCCION |
| `PUT` | `/api/v1/planes-produccion/{idPlan}/detalles/{id}` | Editar la cantidad planificada | ADMIN, PRODUCCION |
| `PATCH` | `/api/v1/planes-produccion/{idPlan}/detalles/{id}/producido` | Registrar cantidad producida | ADMIN, PRODUCCION |
| `DELETE` | `/api/v1/planes-produccion/{idPlan}/detalles/{id}` | Quitar un producto | ADMIN, PRODUCCION |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Dos operaciones distintas sobre la misma fila

Se separan deliberadamente en endpoints diferentes porque ocurren en momentos y con
permisos distintos:

| Operación | Endpoint | Cuándo | Estado del plan |
|---|---|---|---|
| Planificar | `PUT .../detalles/{id}` | Antes de producir | Solo `PLANIFICADO` |
| Registrar producido | `PATCH .../detalles/{id}/producido` | Durante la producción | `EN_PROCESO` |

Un único `PUT` que aceptara ambos campos permitiría modificar lo planificado *después* de
producir, y HU-26 compararía contra un objetivo reescrito a posteriori: la eficiencia
siempre saldría perfecta.

### R-02 · `cantidad_planificada` solo se edita mientras el plan está `PLANIFICADO`
Es la regla que da sentido a HU-26: el plan es el compromiso, y comparar lo producido contra
un compromiso ajustado sobre la marcha no mide nada.

### R-03 · `cantidad_producida` solo se registra con el plan `EN_PROCESO`
No antes —no se ha empezado— ni después de `COMPLETADO` o `CANCELADO`.

### R-04 · Se puede producir más de lo planificado
No se bloquea. Una tanda puede rendir más de lo previsto, y eso es un dato real que HU-26
debe poder mostrar.

La interfaz lo señala (avance > 100%) pero no lo impide. Bloquearlo obligaría a falsear el
registro.

### R-05 · Un producto no puede repetirse en el mismo plan
`UNIQUE (id_plan, id_producto)` añadido en V2 §C-13. Para producir más del mismo producto se
**edita la cantidad planificada**, no se agrega otra línea.

Sin la restricción, HU-22 tendría dos líneas de detalle para el mismo producto y el consumo
teórico se calcularía dos veces.

### R-06 · Un producto sin receta se puede planificar, pero no calcular su consumo
HU-22 calcula el consumo teórico a partir de la receta (HU-12). Un producto sin receta
puede añadirse al plan —quizá se produce artesanalmente— pero **no tendrá consumo teórico**
y quedará fuera del análisis de eficiencia de HU-26.

Se permite, y se advierte claramente en la interfaz. Ocultar el hueco haría que apareciera
después como un consumo teórico de cero sin explicación.

### R-07 · Registrar producción **no** crea el lote ni suma stock
Actualizar `cantidad_producida` es un registro de avance. **No genera el lote de producto
terminado** (HU-19) ni suma a `inventario_producto_terminado`.

Es la consecuencia de HU-19 R-07 y HU-20 R-07: producir y registrar el lote son acciones
separadas en este alcance. **Nada impide completar un plan y olvidar registrar el lote**, con
lo que el producto fabricado no aparecería como stock vendible.

Se documenta como el hueco más visible del flujo de producción y candidato claro a una
historia futura.

### R-08 · Quitar una línea con consumos registrados
Si HU-22 ya registró consumo contra esa línea, quitarla dejaría movimientos de inventario
huérfanos. Se devuelve `409 DETALLE_CON_CONSUMOS`: primero hay que resolver el consumo.

---

## 4. Modelo de datos

**Tabla `detalle_plan_produccion`** (V2 añade el `UNIQUE` y dos `CHECK`)

| Columna | Tipo | Notas |
|---|---|---|
| `id_detalle_plan` | `int unsigned` PK AI | |
| `id_plan` | `int unsigned` NOT NULL FK | CA-01 |
| `id_producto` | `int unsigned` NOT NULL FK | CA-02 |
| `cantidad_planificada` | `decimal(10,2)` NOT NULL | CA-03 · `CHECK > 0` |
| `cantidad_producida` | `decimal(10,2)` NOT NULL **DEFAULT 0.00** | CA-04 · `CHECK >= 0` |

**`UNIQUE KEY uk_detalle_plan (id_plan, id_producto)`** — V2 §C-13.

```json
// POST /api/v1/planes-produccion/9/detalles
{ "idProducto": 3, "cantidadPlanificada": 90.00 }

// 201 Created
{ "idDetallePlan": 21,
  "producto": { "idProducto": 3, "nombre": "Pan francés", "unidad": "und" },
  "cantidadPlanificada": 90.00, "cantidadProducida": 0.00,
  "avance": 0.00, "tieneReceta": true,
  "consumoTeoricoEstimado": [
    { "materiaPrima": "Harina de trigo", "cantidad": 15.000, "unidad": "kg" },
    { "materiaPrima": "Levadura", "cantidad": 0.150, "unidad": "kg" }
  ]
}

// PATCH /api/v1/planes-produccion/9/detalles/21/producido
{ "cantidadProducida": 60.00 }

// 200 OK
{ "idDetallePlan": 21, "cantidadPlanificada": 90.00,
  "cantidadProducida": 60.00, "avance": 66.67 }
```

`consumoTeoricoEstimado` se calcula con la fórmula de HU-12 R-02:
`(cantidad_planificada / receta.cantidad_producir) × detalle_receta.cantidad`.
Es informativo aquí; el consumo real se registra en HU-22.

**Campos ordenables:** `producto`, `cantidadPlanificada`, `avance`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `PLAN_INEXISTENTE` | 404 | El plan no existe |
| `PLAN_NO_EDITABLE` | 409 | El plan no está `PLANIFICADO` (para editar lo planificado) |
| `PLAN_NO_EN_PROCESO` | 409 | El plan no está `EN_PROCESO` (para registrar producción) |
| `PRODUCTO_INEXISTENTE` | 409 | El producto no existe |
| `PRODUCTO_DUPLICADO` | 409 | Ese producto ya está en el plan (R-05) |
| `DETALLE_CON_CONSUMOS` | 409 | La línea tiene consumos registrados (R-08) |
| `CANTIDAD_INVALIDA` | 400 | Cantidad planificada menor o igual a cero |

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

### Por qué `consumoTeoricoEstimado` aparece ya en esta historia

Mostrar el consumo teórico al planificar permite dos cosas que el flujo necesita:

1. **Verificar que hay materia prima suficiente** antes de iniciar la producción, en lugar
   de descubrirlo a mitad de tanda.
2. **Detectar errores en la receta** (HU-12 R-02): si planificar 90 panes arroja un consumo
   teórico de 450 kg de harina, la `cantidad_producir` de la receta está mal interpretada.
   Es una segunda oportunidad de encontrar ese error antes de que llegue a HU-23.
