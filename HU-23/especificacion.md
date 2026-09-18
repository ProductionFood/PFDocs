# HU-23 · Actualización automática de inventario por producción — Especificación

**Fase 7** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** que al registrar consumo de materia prima se actualice automáticamente el inventario,
> **para que** los datos de stock sean consistentes.

**Depende de:** [HU-22](../HU-22/), [HU-10](../HU-10/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Al registrar un `consumo_materia_prima`, se descuenta `cantidad_real` del `inventario` | `InventarioService.registrarSalida()` en la misma transacción |
| CA-02 | Se actualiza la `fecha_actualizacion` en `inventario` | `ON UPDATE CURRENT_TIMESTAMP` **añadido en V2** §C-07 — se actualiza sola |
| CA-03 | Si el stock queda por debajo del `stock_minimo`, se genera una alerta (flag en la respuesta) | Campo `alertas[]` en la respuesta de HU-22 |
| CA-04 | La operación es **transaccional** (si falla, se revierte todo) | `@Transactional` sobre el método de servicio completo |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `POST` | `/api/v1/detalles-plan/{id}/consumos` | *(HU-22)* Registrar consumo · **este es el disparador** | ADMIN, PRODUCCION |
| `PUT` | `/api/v1/consumos/{id}` | *(HU-22)* Corregir consumo · ajusta el inventario | ADMIN, PRODUCCION |
| `GET` | `/api/v1/inventario/alertas` | Materias primas bajo mínimo tras la última producción | ADMIN, PRODUCCION, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Esta historia no tiene endpoints propios

Es el **comportamiento** de HU-22, no una funcionalidad separada. Se documenta aparte porque
concentra las garantías de consistencia del inventario, que son transversales.

**Se implementa junto con HU-22, en la misma transacción y el mismo PR.**

### R-02 · 🔑 La transaccionalidad es el corazón de la historia (CA-04)

El registro de consumo hace cinco cosas que deben ocurrir todas o ninguna:

1. `SELECT ... FOR UPDATE` sobre la fila de inventario;
2. validar stock suficiente;
3. `UPDATE inventario` restando `cantidad_real`;
4. `INSERT` en `movimientos_inventario_mp` con tipo `SALIDA_PRODUCCION`;
5. `INSERT` en `consumo_materia_prima`.

**Un fallo en el paso 5 debe revertir los pasos 3 y 4.** Si no, el inventario baja pero el
consumo no queda registrado: la materia prima desaparece del sistema sin justificación, y el
kardex apunta a un consumo inexistente.

Con **varios ingredientes en una sola petición**, la transacción envuelve todos: si el
tercero no tiene stock, los dos primeros **no** quedan descontados. Una producción a medias
deja un inventario que nadie puede reconstruir.

### R-03 · `SELECT ... FOR UPDATE` no es opcional

Dos producciones simultáneas que consuman la misma materia prima leen el mismo saldo, ambas
validan contra él y ambas descuentan. El resultado es un saldo que no corresponde a ninguna
de las dos operaciones.

El bloqueo se toma **antes** de validar y se libera al cerrar la transacción.
El `CHECK (cantidad_disponible >= 0)` de V2 es la última red: si llega a dispararse, el
bloqueo no está funcionando.

### R-04 · La alerta de stock bajo es informativa, no bloqueante (CA-03)

Si tras el descuento `cantidad_disponible < stock_minimo`, la respuesta incluye una alerta.
**No impide la operación**: la materia prima ya se consumió físicamente, negarse a
registrarlo solo haría que el sistema se alejara de la realidad.

```json
"alertas": [
  { "idMateriaPrima": 5, "nombre": "Harina de trigo",
    "cantidadDisponible": 46.30, "stockMinimo": 50.00, "faltante": 3.70,
    "nivel": "BAJO" }
]
```

Niveles: `BAJO` (por debajo del mínimo) y `AGOTADO` (en cero). Alimenta HU-27 y el dashboard
de HU-24.

### R-05 · Stock insuficiente: se rechaza el registro

Si `cantidad_real > cantidad_disponible` → `409 STOCK_INSUFICIENTE`, y **no se registra
nada**.

Es una decisión discutible: físicamente, si el productor usó 20 kg, los usó, aunque el
sistema creyera que había 15. Pero permitir el registro dejaría el inventario en negativo,
que es un estado sin significado y que el `CHECK` de la base rechazaría de todos modos.

**La discrepancia se resuelve con un ajuste de inventario explícito** —reconociendo que
había más de lo registrado— y luego se registra el consumo. Así queda constancia de que el
inventario estaba mal, en lugar de disimularlo.

### R-06 · `fecha_actualizacion` se mantiene sola (CA-02)

`DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`, añadido en V2 §C-07. El servicio
**no** la asigna: la base lo hace en cada `UPDATE`.

Una fecha que dependa de que el programador se acuerde en cada ruta de código acaba
desactualizada justo en la ruta que alguien olvidó.

### R-07 · Toda salida pasa por `InventarioService`

Ningún servicio ejecuta `UPDATE inventario` directamente. La regla se establece en HU-09 y
esta historia es su principal consumidora.

Es lo que garantiza que **el kardex nunca se desincronice del saldo**: el movimiento y la
actualización viven en el mismo método, y no hay forma de hacer uno sin el otro.

### R-08 · Redondeo de 3 a 2 decimales
`consumo_materia_prima.cantidad_real` es `decimal(10,3)`;
`inventario.cantidad_disponible` es `decimal(10,2)`.

**El descuento redondea con `HALF_UP` a 2 decimales**, siempre igual. Un criterio
inconsistente acumula gramos en cada producción y descuadra el inventario a las semanas, sin
que ningún movimiento individual parezca incorrecto.

---

## 4. Modelo de datos

No introduce tablas nuevas. Coordina tres:

| Tabla | Operación |
|---|---|
| `consumo_materia_prima` | `INSERT` (HU-22) |
| `inventario` | `UPDATE cantidad_disponible` · `fecha_actualizacion` automática |
| `movimientos_inventario_mp` | `INSERT` con `SALIDA_PRODUCCION` (kardex, V2 §C-04) |

```json
// POST /api/v1/detalles-plan/21/consumos  (varios ingredientes a la vez)
{
  "consumos": [
    { "idMateriaPrima": 5, "cantidadReal": 16.200 },
    { "idMateriaPrima": 7, "cantidadReal":  0.150 },
    { "idMateriaPrima": 9, "cantidadReal":  0.300 }
  ]
}

// 201 Created
{
  "consumosRegistrados": 3,
  "movimientos": [
    { "materiaPrima": "Harina de trigo", "descontado": 16.20,
      "saldoAnterior": 112.50, "saldoPosterior": 96.30 },
    { "materiaPrima": "Levadura", "descontado": 0.15,
      "saldoAnterior": 2.00, "saldoPosterior": 1.85 },
    { "materiaPrima": "Sal", "descontado": 0.30,
      "saldoAnterior": 8.20, "saldoPosterior": 7.90 }
  ],
  "alertas": [
    { "idMateriaPrima": 7, "nombre": "Levadura",
      "cantidadDisponible": 1.85, "stockMinimo": 5.00,
      "faltante": 3.15, "nivel": "BAJO" }
  ]
}

// 409 STOCK_INSUFICIENTE — nada se registró
{ "code": "STOCK_INSUFICIENTE",
  "message": "No hay suficiente Levadura. Disponible: 0,15 kg. Requerido: 2,00 kg.",
  "detalle": { "idMateriaPrima": 7, "disponible": 0.150, "requerido": 2.000 } }
```

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `STOCK_INSUFICIENTE` | 409 | La cantidad real supera el inventario disponible (R-05) |
| `INVENTARIO_NO_INICIALIZADO` | 500 | La materia prima no tiene fila en `inventario` — defecto de HU-08 R-01 |

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

### Esta historia es donde se comprueba si el inventario significa algo

Es la contrapartida de la recepción de compra (HU-13 R-01): allí entra el stock, aquí sale.
Entre las dos sostienen todo el control de materias primas.

La regla de conciliación de `00-base/05-ESTANDARES-QA.md` §5 —el `saldo_posterior` del último
movimiento debe ser igual a `cantidad_disponible`— es la prueba de que ambas funcionan. En
cuanto devuelve una fila, alguna ruta de código actualizó el saldo sin registrar el
movimiento, y el inventario deja de ser explicable.
