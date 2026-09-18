# HU-22 · Registro de consumo de materia prima — Especificación

**Fase 7** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** productor,
> **quiero** registrar el consumo real de materia prima por cada producto del plan,
> **para que** compare con lo teórico y mida eficiencia.

**Depende de:** [HU-21](../HU-21/), [HU-12](../HU-12/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `id_detalle_plan` debe existir | Validado; el plan debe estar `EN_PROCESO` |
| CA-02 | `id_materia_prima` debe existir | Validado contra `MateriaPrimaRepository` |
| CA-03 | `cantidad_teorica` obligatoria (viene de la receta) | **La calcula el backend**, no la envía el cliente |
| CA-04 | `cantidad_real` obligatoria (lo que realmente se usó) | **`NOT NULL` desde V2** §C-08 — era nullable |
| CA-05 | `fecha_registro` se registra automáticamente | `DEFAULT CURRENT_TIMESTAMP` ya en el esquema |
| CA-06 | Calcular la eficiencia `(teorica / real) × 100` | ⚠️ Con protección contra división por cero — V2 §C-09 |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/planes-produccion/{idPlan}/consumos` | Consumos de un plan | ADMIN, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/detalles-plan/{idDetallePlan}/consumo-teorico` | Consumo teórico calculado | ADMIN, PRODUCCION |
| `POST` | `/api/v1/detalles-plan/{idDetallePlan}/consumos` | Registrar consumo real · **descuenta inventario (HU-23)** | ADMIN, PRODUCCION |
| `PUT` | `/api/v1/consumos/{id}` | Corregir la cantidad real | ADMIN, PRODUCCION |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · 🔑 `cantidad_teorica` la calcula el servidor, no el cliente

CA-03 dice "viene de la receta". Si el cliente la enviara, podría mandar cualquier número y
la eficiencia de HU-26 sería una cifra inventada.

El backend la calcula con `CalculadoraConsumoService` (creado en HU-21):

```
cantidad_teorica = (cantidad_planificada / receta.cantidad_producir) × detalle_receta.cantidad
```

El cuerpo de la petición **solo trae `idMateriaPrima` y `cantidadReal`**.

### R-02 · ⚠️ `cantidad_teorica` usa lo **planificado**, no lo producido

Decisión que las historias no fijaban y que cambia por completo el significado del reporte.

- Contra lo **planificado**: mide desviación respecto al objetivo. Si se planificaron 90
  panes y se hicieron 60, el consumo teórico de 90 panes será mucho mayor que el real y la
  eficiencia saldrá baja — aunque el consumo por pan haya sido perfecto.
- Contra lo **producido**: mide la eficiencia real del proceso.

**Se calcula contra lo planificado**, porque es lo que CA-03 dice literalmente ("viene de la
receta", y la receta se aplica sobre la cantidad planificada del detalle del plan).

**HU-26 debe mostrar ambas cifras** para que el dato sea interpretable: la desviación contra
el plan y el rendimiento por unidad efectivamente producida. Sin esa distinción, una
eficiencia del 66% no dice si se desperdició materia prima o simplemente se produjo menos.

### R-03 · 🔴 La fórmula de CA-06 divide por cero

`(cantidad_teorica / cantidad_real) × 100` revienta cuando `cantidad_real = 0`, y ese caso
es perfectamente posible: se planificó un producto y no se consumió nada de ese ingrediente.

**MySQL devuelve `NULL` en lugar de lanzar error** (`02-CORRECCIONES-DB.md` §C-09), así que
el fallo es silencioso: el reporte de HU-26 muestra huecos sin explicación.

Protección en tres capas:
- SQL: `CASE WHEN cantidad_real = 0 THEN NULL ELSE ... END`;
- servicio: `BigDecimal`, devolviendo `null` si el divisor es cero;
- frontend: muestra `—`, nunca `NaN` ni `Infinity`.

### R-04 · La fórmula de CA-06 es contraintuitiva

Con `(teorica / real) × 100`, **consumir menos de lo teórico da más de 100%**. Un 200% no
significa el doble de bueno: significa que se usó la mitad del ingrediente previsto, lo cual
casi siempre indica un error de registro o una receta mal calibrada, no una hazaña.

Se implementa CA-06 tal como está escrito, **y además** se expone la métrica interpretable:

```
desviacion% = ((real − teorica) / teorica) × 100
```

Positiva = se gastó de más. Negativa = de menos. Cero = exacto.

HU-26 muestra las dos. La eficiencia porque la pide el criterio de aceptación; la desviación
porque es la que se puede leer.

### R-05 · Registrar consumo descuenta inventario (HU-23)
Es el disparador de HU-23: al registrar, se descuenta `cantidad_real` del inventario de
materia prima, en la misma transacción, con movimiento de kardex.

Las dos historias son inseparables: HU-22 es el registro, HU-23 el efecto. Se implementan
juntas.

### R-06 · Un ingrediente no se registra dos veces por línea de plan
`UNIQUE (id_detalle_plan, id_materia_prima)` añadido en V2 §C-13. Sin él, el inventario se
descontaría dos veces por el mismo consumo.

Para corregir, se **edita** el consumo existente (`PUT /consumos/{id}`), que ajusta el
inventario por diferencia.

### R-07 · Corregir un consumo ajusta el inventario por diferencia
Igual criterio que HU-17 R-07:
- de 15 a 18 kg → se descuentan **3** más;
- de 15 a 12 kg → se **devuelven 3** al inventario.

No se revierte todo y se vuelve a descontar: el kardex debe reflejar hechos, no operaciones
contables intermedias.

### R-08 · Se permite registrar materia prima que no está en la receta
Puede haber consumos imprevistos. En ese caso `cantidad_teorica = 0` y la eficiencia queda
`NULL` (R-03) — con divisor válido pero dividendo cero, la fórmula da 0%.

La interfaz lo marca como **consumo fuera de receta**, que es información valiosa: indica
que la receta no refleja el proceso real.

### R-09 · Precisión decimal
`cantidad_teorica` y `cantidad_real` son `decimal(10,3)` (3 decimales), pero
`inventario.cantidad_disponible` es `decimal(10,2)`.

**Al descontar hay que redondear**, y el criterio debe ser uniforme: `HALF_UP` a 2 decimales.
Un redondeo inconsistente acumula diferencias de gramos que con el tiempo descuadran el
inventario sin causa aparente.

---

## 4. Modelo de datos

**Tabla `consumo_materia_prima`** (V2: `cantidad_real` pasa a `NOT NULL`, más `UNIQUE` y `CHECK`)

| Columna | Tipo | Notas |
|---|---|---|
| `id_consumo` | `int unsigned` PK AI | |
| `id_detalle_plan` | `int unsigned` NOT NULL FK | CA-01 |
| `id_materia_prima` | `int unsigned` NOT NULL FK | CA-02 |
| `cantidad_teorica` | `decimal(10,3)` NOT NULL | CA-03 · **calculada por el servidor** (R-01) |
| `cantidad_real` | `decimal(10,3)` **NOT NULL** | CA-04 · **corregido en V2 §C-08** |
| `fecha_registro` | `datetime` NOT NULL DEFAULT `CURRENT_TIMESTAMP` | CA-05 |

**`UNIQUE KEY uk_consumo_mp (id_detalle_plan, id_materia_prima)`** — V2 §C-13.

```json
// GET /api/v1/detalles-plan/21/consumo-teorico
{ "idDetallePlan": 21, "producto": "Pan francés",
  "cantidadPlanificada": 90.00, "recetaRinde": 30.00, "factor": 3.0,
  "ingredientes": [
    { "idMateriaPrima": 5, "nombre": "Harina de trigo", "unidad": "kg",
      "porTanda": 5.000, "teorico": 15.000, "disponible": 112.50 },
    { "idMateriaPrima": 7, "nombre": "Levadura", "unidad": "kg",
      "porTanda": 0.050, "teorico": 0.150, "disponible": 0.00 }
  ]
}

// POST /api/v1/detalles-plan/21/consumos
{ "idMateriaPrima": 5, "cantidadReal": 16.200 }

// 201 Created
{ "idConsumo": 31,
  "materiaPrima": { "idMateriaPrima": 5, "nombre": "Harina de trigo", "unidad": "kg" },
  "cantidadTeorica": 15.000, "cantidadReal": 16.200,
  "eficiencia": 92.59,
  "desviacionPorcentaje": 8.00,
  "fueraDeReceta": false,
  "fechaRegistro": "2026-09-18T10:22:41",
  "inventario": { "anterior": 112.50, "posterior": 96.30 }
}
```

Nótese que el cuerpo del `POST` **no incluye `cantidadTeorica`** (R-01).

`eficiencia = (15 / 16,2) × 100 = 92,59` · `desviacion = ((16,2 − 15) / 15) × 100 = 8,00`.
La segunda se lee sola: se gastó un 8% más de lo previsto.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `DETALLE_PLAN_INEXISTENTE` | 404 | El `idDetallePlan` no existe |
| `PLAN_NO_EN_PROCESO` | 409 | El plan no está `EN_PROCESO` |
| `MATERIA_PRIMA_INEXISTENTE` | 409 | La materia prima no existe |
| `CONSUMO_DUPLICADO` | 409 | Ese ingrediente ya tiene consumo en esta línea (R-06) |
| `STOCK_INSUFICIENTE` | 409 | No hay inventario suficiente (HU-23) |
| `CANTIDAD_INVALIDA` | 400 | `cantidadReal` negativa |

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

### El caso de `cantidad_real = 0`

```json
// POST con cantidadReal = 0
{ "idConsumo": 32, "cantidadTeorica": 0.150, "cantidadReal": 0.000,
  "eficiencia": null,                    // ← división por cero evitada (R-03)
  "desviacionPorcentaje": -100.00,       // ← se consumió 100% menos de lo previsto
  "advertencia": "No se registró consumo de este ingrediente" }
```

`eficiencia: null` se muestra como `—` en la interfaz. `desviacion` sigue siendo calculable
porque divide entre `cantidad_teorica`, que es distinta de cero.

Este ejemplo muestra por qué R-04 importa: la eficiencia no se puede calcular, pero la
desviación dice exactamente lo que pasó.
