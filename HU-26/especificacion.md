# HU-26 · Eficiencia de producción — Especificación

**Fase 8** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** ver la eficiencia de producción (planificado vs. producido, teórico vs. real),
> **para que** evalúe el rendimiento de la panadería.

**Depende de:** [HU-22](../HU-22/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Cada plan con sus productos: `cantidad_planificada` vs. `cantidad_producida` | Desde `detalle_plan_produccion` |
| CA-02 | Consumo teórico vs. real por materia prima | Desde `consumo_materia_prima` |
| CA-03 | Porcentaje de eficiencia general | ⚠️ Agregado, con protección contra división por cero |
| CA-04 | Filtrar por rango de fechas de producción | `?fechaInicio=&fechaFin=` sobre `planes_produccion.fecha_produccion` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/reportes/eficiencia` | Reporte por período | ADMIN, PRODUCCION |
| `GET` | `/api/v1/reportes/eficiencia/{idPlan}` | Detalle de un plan | ADMIN, PRODUCCION |
| `GET` | `/api/v1/reportes/eficiencia/materias-primas` | Desviación acumulada por insumo | ADMIN, PRODUCCION, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · 🔑 Son **dos** eficiencias distintas y hay que separarlas

CA-01 y CA-02 miden cosas diferentes, y mezclarlas produce un número sin significado:

| Indicador | Fórmula | Qué dice |
|---|---|---|
| **Cumplimiento del plan** | `producido / planificado × 100` | ¿Se fabricó lo que se prometió? |
| **Rendimiento de materia prima** | `teórica / real × 100` | ¿Se gastó lo previsto por unidad? |

Un plan puede tener 60% de cumplimiento (se hicieron 60 de 90 panes) y 100% de rendimiento
(cada pan consumió exactamente lo que debía). Son dos problemas distintos: el primero es de
planificación o de capacidad, el segundo es de proceso.

**El reporte muestra ambas por separado y nunca las promedia entre sí.**

### R-02 · ⚠️ La eficiencia de materia prima está sesgada por lo planificado

Consecuencia directa de HU-22 R-02: `cantidad_teorica` se calcula sobre la **cantidad
planificada**, no sobre la producida.

Si se planificaron 90 panes y se hicieron 60, el teórico corresponde a 90 panes y el real a
60: la eficiencia sale artificialmente alta, como si se hubiera ahorrado materia prima.

**Por eso el reporte expone una tercera métrica, la única comparable entre planes:**

```
rendimiento_ajustado = (teórica × producido / planificado) / real × 100
```

Es decir, se ajusta el teórico a lo que realmente se produjo antes de comparar. Con 60 de 90
panes y consumo proporcional, esta métrica da 100% — que es la verdad.

**Sin este ajuste, comparar la eficiencia de dos planes con distinto grado de cumplimiento
no significa nada.** Es la corrección más importante de este reporte.

### R-03 · 🔴 División por cero en tres lugares

| Cálculo | Divisor | Cuándo es cero |
|---|---|---|
| Cumplimiento | `cantidad_planificada` | Nunca — `CHECK > 0` en V2 |
| Rendimiento | `cantidad_real` | Se registró consumo cero |
| Desviación | `cantidad_teorica` | Ingrediente fuera de receta (HU-22 R-08) |

**MySQL devuelve `NULL` al dividir por cero, sin error** (§C-09). Los dos últimos casos son
reales y se protegen con `CASE WHEN ... THEN NULL`; el frontend muestra `—`.

**Un `NULL` no se puede promediar como si fuera cero.** Al agregar, se excluyen de la media
y se informa cuántos registros quedaron fuera; si no, el promedio de eficiencia baja
artificialmente cada vez que alguien registra un consumo en cero.

### R-04 · La desviación es la métrica que la gente entiende

Repetido de HU-22 R-04 porque aquí es donde se consume:

```
desviación% = (real − teórica) / teórica × 100
```

Positiva = se gastó de más. Es directamente legible, a diferencia de una eficiencia del 200%
que casi todo el mundo interpreta al revés.

El reporte encabeza con la desviación y ofrece la eficiencia como columna secundaria.

### R-05 · Solo se reportan planes con consumo registrado
Un plan `PLANIFICADO` sin consumos no tiene nada que medir. Se incluyen los `EN_PROCESO`,
`COMPLETADO` y `CANCELADO` **que tengan al menos un consumo registrado**.

Los cancelados se marcan aparte: su cumplimiento bajo es esperable y promediarlos con el
resto distorsiona el indicador general.

### R-06 · La desviación acumulada por materia prima es el hallazgo más útil
Agregando todos los planes del período por insumo se detecta el patrón: si la harina lleva
un +7% sostenido durante un mes, o la receta está mal calibrada o hay desperdicio
sistemático. Un plan aislado no revela eso; treinta sí.

Es el endpoint `/eficiencia/materias-primas` y probablemente lo más accionable del reporte.

### R-07 · Los productos sin receta quedan fuera del análisis
Sin receta no hay consumo teórico (HU-21 R-06), así que no hay nada contra qué comparar.
Se listan aparte para que el hueco sea visible, no se omiten en silencio.

---

## 4. Modelo de datos

Sin cambios de esquema. Consulta `planes_produccion`, `detalle_plan_produccion`,
`consumo_materia_prima`, `productos` y `materias_primas`.
V2 añadió `idx_planes_fecha (fecha_produccion, estado)`.

```json
// GET /api/v1/reportes/eficiencia?fechaInicio=2026-09-01&fechaFin=2026-09-30
{
  "periodo": { "desde": "2026-09-01", "hasta": "2026-09-30" },
  "resumen": {
    "planesAnalizados": 12,
    "planesCancelados": 2,
    "cumplimientoPromedio": 87.40,
    "rendimientoPromedio": 94.20,
    "rendimientoAjustadoPromedio": 96.80,
    "registrosSinRendimiento": 3,
    "desviacionMateriaPrima": 5.80
  },
  "planes": [
    { "idPlan": 9, "fechaProduccion": "2026-09-18", "estado": "COMPLETADO",
      "responsable": "Ismael Batalla",
      "cumplimiento": 66.67,
      "rendimiento": 92.59,
      "rendimientoAjustado": 61.73,
      "productos": [
        { "producto": "Pan francés", "planificada": 90.00, "producida": 60.00,
          "cumplimiento": 66.67 }
      ],
      "consumos": [
        { "materiaPrima": "Harina de trigo", "teorica": 15.000, "real": 16.200,
          "desviacion": 8.00, "eficiencia": 92.59 },
        { "materiaPrima": "Sal", "teorica": 0.300, "real": 0.000,
          "desviacion": -100.00, "eficiencia": null }
      ]
    }
  ],
  "sinReceta": [
    { "producto": "Pan integral", "planes": 2,
      "nota": "Sin receta: no se puede calcular consumo teórico" }
  ]
}

// GET /api/v1/reportes/eficiencia/materias-primas?fechaInicio=2026-09-01&fechaFin=2026-09-30
[
  { "materiaPrima": "Harina de trigo", "unidad": "kg",
    "teoricaAcumulada": 180.000, "realAcumulada": 192.600,
    "desviacion": 7.00, "planes": 12,
    "tendencia": "CONSUMO_EXCESIVO" },
  { "materiaPrima": "Levadura", "unidad": "kg",
    "teoricaAcumulada": 1.800, "realAcumulada": 1.795,
    "desviacion": -0.28, "planes": 12,
    "tendencia": "NORMAL" }
]
```

`eficiencia: null` cuando `real = 0` (R-03). `rendimientoAjustado` es la métrica comparable
entre planes (R-02).

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `RANGO_FECHAS_INVALIDO` | 400 | `fechaInicio` posterior a `fechaFin` |
| `PLAN_INEXISTENTE` | 404 | El plan no existe |
| `PLAN_SIN_CONSUMOS` | 404 | El plan no tiene consumos registrados (R-05) |

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

### Por qué este reporte es difícil de hacer bien

Tres trampas, y ninguna produce un error visible:

1. **Promediar `NULL` como cero** (R-03) hunde el indicador general cada vez que alguien
   registra un consumo en cero.
2. **Comparar eficiencias de planes con distinto cumplimiento** sin ajustar (R-02) lleva a
   conclusiones invertidas: el plan que menos produjo parece el más eficiente.
3. **Mezclar cumplimiento y rendimiento** en un solo "porcentaje de eficiencia" (R-01)
   produce una cifra que no responde a ninguna pregunta.

CA-03 pide "el porcentaje de eficiencia general". Se entrega, pero acompañado de las
métricas que lo hacen interpretable.
