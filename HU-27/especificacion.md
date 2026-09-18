# HU-27 · Alerta de stock bajo — Especificación

**Fase 8** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** ver materias primas con stock por debajo del mínimo,
> **para que** reabastezca a tiempo.

**Depende de:** [HU-10](../HU-10/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Listar todas las materias primas donde `cantidad_disponible < stock_minimo` | Vista `v_inventario_materia_prima` (V2), `estado_stock IN ('BAJO','AGOTADO')` |
| CA-02 | Mostrar nombre, stock actual y stock mínimo | Campos de la vista |
| CA-03 | Mostrar la diferencia (cuánto falta para alcanzar el mínimo) | Campo `faltante` — `GREATEST(minimo − disponible, 0)` |
| CA-04 | Ordenar por urgencia (mayor diferencia primero) | `ORDER BY faltante DESC` por defecto |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/inventario/alertas` | Materias primas bajo mínimo | ADMIN, PRODUCCION, COMPRAS, CONSULTA |
| `GET` | `/api/v1/inventario/alertas/resumen` | Conteo para el indicador global | Todos los roles autenticados |
| `GET` | `/api/v1/inventario/alertas/sugerencia-compra` | Propuesta de reposición agrupada por proveedor | ADMIN, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · La historia más simple de la fase, y la que más se usa

Una sola consulta sobre la vista creada en V2. Toda la complejidad está resuelta allí:
el `LEFT JOIN`, el `COALESCE`, el `CASE` de `estado_stock` y el `GREATEST` del faltante.

Es también el reporte que se consulta a diario: sostiene el indicador global de la barra
superior y la tarjeta del dashboard (HU-24 CA-03).

### R-02 · "Por debajo del mínimo" es estricto (CA-01)

`cantidad_disponible < stock_minimo`. Con disponible **exactamente igual** al mínimo,
**no hay alerta**: se está justo en el nivel definido como aceptable.

Es un borde que conviene tener claro porque se presta a interpretaciones: `<` no es `<=`.

### R-03 · `AGOTADO` y `BAJO` son urgencias distintas

| Nivel | Condición | Qué significa |
|---|---|---|
| `AGOTADO` | `disponible = 0` | **Bloquea la producción.** No se puede fabricar nada que lo use |
| `BAJO` | `0 < disponible < minimo` | Hay que reponer, pero aún se puede trabajar |

`AGOTADO` siempre va primero, independientemente del faltante: una materia prima agotada con
un faltante de 2 kg es más urgente que otra con 40 kg disponibles y un faltante de 60.

**El orden real es: primero `AGOTADO`, luego `BAJO` por faltante descendente.** CA-04 pide
ordenar por diferencia; se cumple dentro de cada nivel.

### R-04 · `stock_minimo = 0` significa "no alertar"
Una materia prima con mínimo cero nunca aparece como `BAJO` (HU-08 R-06). Es la forma de
excluir del monitoreo los insumos que se compran bajo pedido.

Si además su disponible es cero, sí aparece como `AGOTADO`: no hay existencias.

### R-05 · Las materias primas inactivas se excluyen por defecto
Una materia prima desactivada (HU-08 R-04) no se va a reponer. Alertar sobre ella genera
ruido permanente.

Se excluyen salvo que se pida `?incluirInactivas=true`. Pero si tiene stock, sigue
apareciendo en el inventario de HU-10: son dos preguntas distintas — "qué hay" y "qué hay
que comprar".

### R-06 · La sugerencia de compra agrupa por proveedor

El endpoint `/sugerencia-compra` propone qué comprar y a quién, deduciendo el proveedor de
la **última compra recibida** de cada materia prima.

La cantidad sugerida es el faltante más un margen del 20%, redondeado hacia arriba: comprar
exactamente el faltante deja la materia prima justo en el mínimo, y al primer consumo vuelve
a estar en alerta.

Es una sugerencia, no una compra automática: quien compra decide.

### R-07 · Sin notificaciones automáticas
No hay correo ni notificaciones push: no está en los criterios de aceptación ni en el stack.
La alerta se consulta activamente o se ve en el indicador global de la barra superior.

Se registra como mejora posible.

---

## 4. Modelo de datos

Sin cambios de esquema. Consulta la vista `v_inventario_materia_prima` creada en V2.

```json
// GET /api/v1/inventario/alertas
{
  "generadoEn": "2026-09-18T10:42:33",
  "resumen": { "total": 3, "agotadas": 1, "bajas": 2 },
  "items": [
    { "idMateriaPrima": 7, "nombre": "Levadura", "unidad": "kg",
      "cantidadDisponible": 0.00, "stockMinimo": 5.00, "faltante": 5.00,
      "nivel": "AGOTADO", "costoUnitario": 180.00,
      "costoReposicion": 1080.00,
      "ultimaActualizacion": "2026-09-18T09:12:44" },
    { "idMateriaPrima": 12, "nombre": "Mantequilla", "unidad": "kg",
      "cantidadDisponible": 4.50, "stockMinimo": 20.00, "faltante": 15.50,
      "nivel": "BAJO", "costoUnitario": 18500.00,
      "costoReposicion": 344100.00,
      "ultimaActualizacion": "2026-09-17T14:20:10" },
    { "idMateriaPrima": 5, "nombre": "Harina de trigo", "unidad": "kg",
      "cantidadDisponible": 46.30, "stockMinimo": 50.00, "faltante": 3.70,
      "nivel": "BAJO", "costoUnitario": 3200.00,
      "costoReposicion": 14208.00,
      "ultimaActualizacion": "2026-09-18T10:22:41" }
  ],
  "costoReposicionTotal": 359388.00
}

// GET /api/v1/inventario/alertas/resumen
{ "total": 3, "agotadas": 1, "bajas": 2 }

// GET /api/v1/inventario/alertas/sugerencia-compra
[
  { "proveedor": { "idProveedor": 3, "nombre": "Harinas del Caribe S.A.S." },
    "items": [
      { "materiaPrima": "Harina de trigo", "faltante": 3.70,
        "cantidadSugerida": 4.44, "costoEstimado": 14208.00 }
    ],
    "totalEstimado": 14208.00 },
  { "proveedor": { "idProveedor": 5, "nombre": "Distribuidora Lácteos" },
    "items": [
      { "materiaPrima": "Mantequilla", "faltante": 15.50,
        "cantidadSugerida": 18.60, "costoEstimado": 344100.00 }
    ],
    "totalEstimado": 344100.00 }
]
```

El orden de `items` es `AGOTADO` primero, luego `BAJO` por faltante descendente (R-03).
`costoReposicion` usa `materias_primas.costo_unitario`, que es un valor de referencia
(HU-08 R-03).

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `RANGO_INVALIDO` | 400 | Parámetro de filtro mal formado |

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

### El cierre del ciclo de inventario

| Historia | Papel |
|---|---|
| HU-08 | Define el `stock_minimo` de cada materia prima |
| HU-13 | Suma stock al recibir compras |
| HU-23 | Resta stock al producir |
| HU-10 | Muestra el estado actual |
| **HU-27** | **Avisa cuándo actuar** |
| HU-13 | Y el ciclo vuelve a empezar con una compra |

Es la historia que convierte el inventario de un registro pasivo en una herramienta
operativa. Si HU-27 no muestra alertas cuando debería, el problema casi nunca está aquí:
está en que alguna materia prima no tiene fila de inventario (HU-08 R-01) o en que las
compras no están sumando (§C-03).
