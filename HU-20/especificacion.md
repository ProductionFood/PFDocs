# HU-20 · Creación de plan de producción — Especificación

**Fase 7** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** productor,
> **quiero** crear un plan de producción con fechas, estado, observaciones y usuario responsable,
> **para que** organice la fabricación.

**Depende de:** [HU-01](../HU-01/) (usuarios)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `fecha_planificacion` y `fecha_produccion` obligatorias | `@NotNull`; producción no puede ser anterior a planificación |
| CA-02 | `estado` obligatorio | Canónicos: `PLANIFICADO`, `EN_PROCESO`, `COMPLETADO`, `CANCELADO` — `CHECK` en V2 §C-12 |
| CA-03 | `observaciones` opcional, máx. 255 | `@Size(max=255)` |
| CA-04 | `id_usuario` debe existir | Validado; se asigna el usuario autenticado por defecto |
| CA-05 | Se retorna el plan creado con su `id_plan` | `201` + `Location` + `PlanResponse` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/planes-produccion` | Listar con filtros y paginación | ADMIN, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/planes-produccion/{id}` | Obtener plan con su detalle | ADMIN, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/planes-produccion` | Crear un plan | ADMIN, PRODUCCION |
| `PUT` | `/api/v1/planes-produccion/{id}` | Editar cabecera (solo si está `PLANIFICADO`) | ADMIN, PRODUCCION |
| `PATCH` | `/api/v1/planes-produccion/{id}/estado` | Cambiar estado | ADMIN, PRODUCCION |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Máquina de estados

```
PLANIFICADO ──▶ EN_PROCESO ──▶ COMPLETADO
     │               │
     └───────────────┴───────▶ CANCELADO
```

`COMPLETADO` es terminal. A diferencia de compras y pedidos, **ningún cambio de estado aquí
mueve inventario**: el movimiento lo genera el registro de consumo (HU-22 y HU-23), no la
transición del plan.

Transición no permitida → `409 TRANSICION_INVALIDA`.

### R-02 · No se puede pasar a `EN_PROCESO` sin productos
Un plan sin líneas de detalle (HU-21) no tiene nada que producir. Iniciarlo dejaría un plan
en proceso que nunca podrá completarse con sentido → `409 PLAN_SIN_PRODUCTOS`.

### R-03 · `fecha_produccion` no puede ser anterior a `fecha_planificacion`
Se planifica antes de producir. Ambas pueden ser el mismo día.
Se permite `fecha_produccion` **futura**: es lo normal, se planifica para mañana.

### R-04 · El responsable es el usuario autenticado, por defecto
CA-04 exige `id_usuario`. Si el cuerpo no lo envía, se toma de `SecurityUtils.idUsuarioActual()`.
Solo `ADMIN` puede asignar el plan a otro usuario; `PRODUCCION` crea planes a su nombre.

Así el responsable no queda a merced de lo que el cliente decida enviar.

### R-05 · La cabecera solo se edita mientras está `PLANIFICADO`
Una vez `EN_PROCESO` hay consumos registrados contra las líneas del plan; cambiar la fecha
o el responsable desvirtuaría los datos de eficiencia de HU-26.

### R-06 · Cancelar un plan no revierte el consumo ya registrado
Si el plan estaba `EN_PROCESO` con consumos registrados (HU-22), esa materia prima **ya se
usó físicamente**. Cancelar el plan no la devuelve al inventario.

Es deliberado: el material consumido no vuelve al saco. Cancelar significa "no se sigue
produciendo", no "no pasó nada". Para corregir un consumo mal registrado hace falta un
ajuste de inventario explícito.

### R-07 · El plan no genera lotes automáticamente
Lo natural sería que al completar un plan se crearan los lotes de producto terminado
(HU-19). **Las historias no lo piden y no se implementa**: producir y registrar el lote son
acciones separadas.

Se documenta como mejora evidente, igual que en HU-19 R-07. Por ahora nada impide completar
un plan y olvidar el registro del lote, con lo que el producto fabricado no aparecería como
stock vendible.

---

## 4. Modelo de datos

**Tabla `planes_produccion`** (V2 añade el `CHECK` de estado y el índice)

| Columna | Tipo | Notas |
|---|---|---|
| `id_plan` | `int unsigned` PK AI | |
| `fecha_planificacion` | `date` NOT NULL | CA-01 |
| `fecha_produccion` | `date` NOT NULL | CA-01, R-03 |
| `estado` | `varchar(30)` NOT NULL | CA-02 · `CHECK` en V2 §C-12 |
| `observaciones` | `varchar(255)` NULL | CA-03 |
| `id_usuario` | `int unsigned` NOT NULL FK | CA-04, R-04 |

```json
// POST /api/v1/planes-produccion
{ "fechaPlanificacion": "2026-09-18", "fechaProduccion": "2026-09-19",
  "observaciones": "Producción para el pedido #47 y reposición de estante" }

// 201 Created · Location: /api/v1/planes-produccion/9
{ "idPlan": 9,
  "fechaPlanificacion": "2026-09-18", "fechaProduccion": "2026-09-19",
  "estado": "PLANIFICADO",
  "observaciones": "Producción para el pedido #47 y reposición de estante",
  "responsable": { "idUsuario": 4, "nombre": "Ismael Batalla" },
  "cantidadProductos": 0, "editable": true }
```

El cuerpo **no acepta `estado`**: siempre nace `PLANIFICADO`.

**Campos ordenables:** `fechaProduccion`, `fechaPlanificacion`, `estado`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `USUARIO_INEXISTENTE` | 409 | El `idUsuario` no existe |
| `TRANSICION_INVALIDA` | 409 | Transición de estado no permitida (R-01) |
| `PLAN_SIN_PRODUCTOS` | 409 | Se intenta iniciar un plan sin líneas (R-02) |
| `PLAN_NO_EDITABLE` | 409 | Se intenta modificar un plan que no está `PLANIFICADO` (R-05) |
| `FECHAS_INVALIDAS` | 400 | `fechaProduccion` anterior a `fechaPlanificacion` (R-03) |

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

### Cómo encaja con el resto de la fase

| Historia | Qué aporta |
|---|---|
| **HU-20** (esta) | Cabecera del plan: cuándo, quién, en qué estado |
| HU-21 | Qué productos y cuántos: planificado vs. producido |
| HU-22 | Consumo real de materia prima por producto del plan |
| HU-23 | El consumo descuenta inventario automáticamente |

La cabecera no mueve stock. Todo el efecto sobre el inventario ocurre en HU-22 y HU-23.
