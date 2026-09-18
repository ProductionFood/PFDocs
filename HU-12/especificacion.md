# HU-12 · Gestión de recetas — Especificación

**Fase 4** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** asociar una receta a cada producto indicando materias primas y cantidades,
> **para que** sepa qué ingredientes se necesitan por unidad.

**Depende de:** [HU-08](../HU-08/), [HU-11](../HU-11/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Cada producto puede tener **una sola** receta | `UNIQUE KEY id_producto` ya existe en el esquema |
| CA-02 | La receta tiene `nombre`, `cantidad_producir`, `id_unidad` y `estado` | `@NotBlank`, `@NotNull`, FK validada; `DEFAULT 1` añadido en V2 §C-06 |
| CA-03 | El detalle lista materias primas con sus cantidades | Subrecurso `/recetas/{id}/detalles` |
| CA-04 | Agregar, editar y eliminar materias primas del detalle | `POST`, `PUT`, `DELETE` sobre el subrecurso |
| CA-05 | **No se permite duplicar** la misma materia prima en una receta | `UNIQUE (id_receta, id_materia_prima)` **añadido en V2** §C-05 |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/recetas` | Listar recetas con paginación | ADMIN, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/recetas/{id}` | Obtener receta con su detalle completo | ADMIN, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/productos/{idProducto}/receta` | Receta de un producto | ADMIN, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/recetas` | Crear receta (cabecera + detalle inicial) | ADMIN, PRODUCCION |
| `PUT` | `/api/v1/recetas/{id}` | Editar la cabecera | ADMIN, PRODUCCION |
| `PATCH` | `/api/v1/recetas/{id}/estado` | Activar o desactivar | ADMIN, PRODUCCION |
| `POST` | `/api/v1/recetas/{id}/detalles` | Agregar un ingrediente | ADMIN, PRODUCCION |
| `PUT` | `/api/v1/recetas/{id}/detalles/{idDetalle}` | Editar la cantidad de un ingrediente | ADMIN, PRODUCCION |
| `DELETE` | `/api/v1/recetas/{id}/detalles/{idDetalle}` | Quitar un ingrediente | ADMIN, PRODUCCION |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Sin duplicados: la garantía es de la base (CA-05)

El esquema original **no tenía** la restricción, aunque CA-05 la exige. Validarla solo en
el servicio no basta: dos peticiones simultáneas pasan ambas el `SELECT` previo y ambas
insertan.

V2 añadió `UNIQUE KEY uk_detalle_receta (id_receta, id_materia_prima)` (§C-05). El servicio
sigue comprobando para devolver un `409` legible, pero **la base es la que garantiza**.

Un ingrediente duplicado haría que HU-22 calcule el doble de consumo teórico y HU-23
descuente el doble de inventario.

### R-02 · `cantidad_producir` es el rendimiento de la receta

Es la cantidad de producto que rinde **una tanda completa**, no una unidad. Una receta que
produce 30 panes con 5 kg de harina tiene `cantidad_producir = 30`.

**El consumo teórico de HU-22 depende de esto:**

```
consumo_teórico = (cantidad_planificada / cantidad_producir) × cantidad_del_ingrediente
```

Planificar 90 panes con esa receta consume `(90 / 30) × 5 = 15 kg` de harina.

**Si `cantidad_producir` se interpreta como "por unidad" cuando en realidad es por tanda,
todos los cálculos de producción salen multiplicados por 30.** Es el malentendido más
probable de esta historia, y no produce ningún error: solo números absurdos en HU-22 y
HU-26. La interfaz debe hacer explícito el significado.

### R-03 · `cantidad_producir` nunca puede ser cero
Dividir por ella es el núcleo del cálculo de R-02. El `CHECK (cantidad_producir > 0)` y la
validación del servicio lo impiden. Es la misma clase de problema que el divisor cero de
HU-22 (§C-09), pero aquí sí se puede bloquear en el origen.

### R-04 · La unidad de la receta es la de producción
`recetas.id_unidad` es la unidad en que se mide `cantidad_producir`, y puede diferir de la
unidad de venta del producto (HU-11 R-04). Lo habitual es que coincidan, pero el esquema
permite que no.

### R-05 · Una receta sin ingredientes es inútil pero se permite crearla
Se puede guardar la cabecera y agregar el detalle después, porque obligar a completar todo
de una vez hace incómodo el trabajo real.

**Pero una receta sin detalle no sirve para planificar**: HU-22 calcularía consumo teórico
cero. La interfaz la marca como incompleta y HU-21 advierte al usarla.

### R-06 · Las cantidades del detalle llevan 3 decimales
`detalle_receta.cantidad` es `decimal(10,3)`, a diferencia de casi todas las demás
cantidades del sistema, que son `(10,2)`.

Es correcto: en repostería se usan gramos de sal o levadura donde el tercer decimal importa.
**Al calcular el consumo teórico, el resultado se guarda también con 3 decimales**
(`consumo_materia_prima.cantidad_teorica` es `(10,3)`), pero el inventario es `(10,2)`.
Ese redondeo al descontar debe hacerse de forma consistente — ver HU-23.

### R-07 · Se puede eliminar una línea de detalle
Es la única excepción del proyecto a "no se elimina": un ingrediente quitado de una receta
no tiene valor histórico, y `detalle_receta` no es referenciada por ninguna otra tabla.
La cabecera de la receta, en cambio, solo se desactiva.

---

## 4. Modelo de datos

**Tabla `recetas`**

| Columna | Tipo | Notas |
|---|---|---|
| `id_receta` | `int unsigned` PK AI | |
| `id_producto` | `int unsigned` NOT NULL **UNIQUE** FK | CA-01 |
| `nombre` | `varchar(100)` NOT NULL | CA-02 |
| `cantidad_producir` | `decimal(10,2)` NOT NULL | CA-02 · R-02 · `CHECK > 0` |
| `id_unidad` | `int unsigned` NOT NULL FK | CA-02 · unidad de producción (R-04) |
| `estado` | `tinyint(1)` NOT NULL **DEFAULT 1** | V2 §C-06 |

**Tabla `detalle_receta`**

| Columna | Tipo | Notas |
|---|---|---|
| `id_detalle_receta` | `int unsigned` PK AI | |
| `id_receta` | `int unsigned` NOT NULL FK | |
| `id_materia_prima` | `int unsigned` NOT NULL FK | |
| `cantidad` | `decimal(10,3)` NOT NULL | R-06 · `CHECK > 0` |

**`UNIQUE KEY uk_detalle_receta (id_receta, id_materia_prima)`** — añadido en V2 §C-05.

```json
// POST /api/v1/recetas
{
  "idProducto": 3, "nombre": "Masa de pan francés",
  "cantidadProducir": 30.00, "idUnidad": 5,
  "detalles": [
    { "idMateriaPrima": 5, "cantidad": 5.000 },
    { "idMateriaPrima": 7, "cantidad": 0.050 },
    { "idMateriaPrima": 9, "cantidad": 0.100 }
  ]
}

// 201 Created
{
  "idReceta": 2, "producto": { "idProducto": 3, "nombre": "Pan francés" },
  "nombre": "Masa de pan francés", "cantidadProducir": 30.00,
  "unidad": { "abreviatura": "und" }, "activo": true,
  "detalles": [
    { "idDetalleReceta": 11, "materiaPrima": { "idMateriaPrima": 5, "nombre": "Harina de trigo", "unidad": "kg" },
      "cantidad": 5.000, "costoEstimado": 16000.00 },
    { "idDetalleReceta": 12, "materiaPrima": { "idMateriaPrima": 7, "nombre": "Levadura", "unidad": "g" },
      "cantidad": 0.050, "costoEstimado": 9.00 }
  ],
  "costoTotalEstimado": 16009.00,
  "costoPorUnidad": 533.63
}
```

`costoEstimado` usa `materias_primas.costo_unitario` (HU-08 R-03), que es un valor de
referencia. **No es el costo real**, que depende de cada compra.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `PRODUCTO_YA_TIENE_RECETA` | 409 | El producto ya tiene una receta (CA-01) |
| `MATERIA_PRIMA_DUPLICADA` | 409 | Ese ingrediente ya está en la receta (CA-05) |
| `PRODUCTO_INEXISTENTE` | 409 | El `idProducto` no existe o está inactivo |
| `MATERIA_PRIMA_INEXISTENTE` | 409 | El `idMateriaPrima` no existe o está inactiva |
| `CANTIDAD_PRODUCIR_INVALIDA` | 400 | `cantidadProducir` menor o igual a cero (R-03) |

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

### Por qué esta historia sostiene toda la Fase 7

HU-22 calcula el consumo teórico a partir de la receta, y HU-23 descuenta el inventario con
ese cálculo. Un error aquí —un ingrediente duplicado, una `cantidad_producir` mal
interpretada— **no se manifiesta en esta historia**: aparece tres sprints después como un
inventario que se descuenta mal y una eficiencia de producción sin sentido.

Por eso CA-05 tiene respaldo en la base (R-01) y por eso R-02 insiste en dejar explícito el
significado de `cantidad_producir` en la interfaz.
