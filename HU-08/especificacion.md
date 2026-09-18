# HU-08 · Gestión de materias primas — Especificación

**Fase 3** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** CRUD de materias primas (nombre, unidad, stock mínimo, costo unitario, estado),
> **para que** conozca los ingredientes disponibles.

**Depende de:** [HU-07](../HU-07/) (unidades de medida)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Nombre obligatorio, máx. 100 | `@NotBlank` + `@Size(max=100)` |
| CA-02 | `id_unidad` debe existir en `unidades_medida` | Validado en el servicio contra `UnidadMedidaRepository` |
| CA-03 | `stock_minimo` y `costo_unitario` obligatorios (decimales) | `@NotNull` + `@DecimalMin("0.00")`; `BigDecimal` |
| CA-04 | Estado por defecto activo (`1`) | `DEFAULT 1` **añadido en V2** (§C-06) — el esquema original no lo tenía |
| CA-05 | Listar con paginación y filtrar por nombre | `GET /materias-primas?nombre=&page=&size=` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/materias-primas` | Listar con filtro y paginación | ADMIN, PRODUCCION, COMPRAS, CONSULTA |
| `GET` | `/api/v1/materias-primas/{id}` | Obtener una | ADMIN, PRODUCCION, COMPRAS, CONSULTA |
| `POST` | `/api/v1/materias-primas` | Registrar | ADMIN |
| `PUT` | `/api/v1/materias-primas/{id}` | Editar | ADMIN |
| `PATCH` | `/api/v1/materias-primas/{id}/estado` | Activar o desactivar | ADMIN |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Al crear una materia prima se crea su fila de inventario

**Esta es la regla más importante de la historia.** La tabla `inventario` tiene
`UNIQUE (id_materia_prima)`: hay exactamente una fila por materia prima, o ninguna.

Si no se crea al dar de alta la materia prima, ocurre lo siguiente:
- HU-10 la muestra sin cantidad, o la omite del listado;
- HU-27 no puede detectar que está bajo mínimo, porque no hay fila que comparar;
- HU-23 falla al intentar descontar de una fila inexistente.

**Al crear una materia prima, en la misma transacción, se inserta
`inventario (id_materia_prima, cantidad_disponible = 0)`.**

La vista `v_inventario_materia_prima` usa `LEFT JOIN` y `COALESCE(...,0)` como red de
seguridad, pero eso corrige la *consulta*, no el problema: sin fila real no hay dónde sumar
cuando llegue la primera compra.

### R-02 · `stock_minimo` y `costo_unitario` con `BigDecimal`, nunca `double`
Un `double` no representa exactamente `0.1`. Acumular costos con `double` produce
diferencias de centavos que aparecen como descuadres imposibles de explicar
(`04-CONTRATO-API.md` §6).

### R-03 · `costo_unitario` es un valor de referencia, no el costo real
El costo real de cada compra está en `detalle_compra.precio_unitario` (HU-14), que varía
entre proveedores y fechas. Este campo sirve para estimar el costo de una receta antes de
comprar.

**No se actualiza solo** al registrar una compra: hacerlo sobrescribiría el valor de
referencia con el precio puntual de la última compra, que puede ser atípico.

### R-04 · Desactivar no borra el stock
Una materia prima inactiva conserva su inventario. Desactivar significa "no comprar más ni
usar en recetas nuevas", no "desapareció del almacén". HU-10 debe poder mostrarla si aún
tiene existencias.

### R-05 · No se elimina
`inventario`, `lotes`, `detalle_compra`, `detalle_receta` y `consumo_materia_prima` la
referencian. Solo se desactiva.

### R-06 · `stock_minimo` puede ser cero
Significa "no alertar por esta materia prima". Es válido para insumos que se compran bajo
pedido. El `CHECK (stock_minimo >= 0)` de V2 impide negativos, que no tendrían sentido.

---

## 4. Modelo de datos

**Tabla `materias_primas`**

| Columna | Tipo | Notas |
|---|---|---|
| `id_materia_prima` | `int unsigned` PK AI | |
| `nombre` | `varchar(100)` NOT NULL | CA-01 |
| `id_unidad` | `int unsigned` NOT NULL FK | CA-02 |
| `stock_minimo` | `decimal(10,2)` NOT NULL | CA-03 · `CHECK >= 0` (V2) |
| `costo_unitario` | `decimal(10,2)` NOT NULL | CA-03 · `CHECK >= 0` (V2) |
| `estado` | `tinyint(1)` NOT NULL **DEFAULT 1** | CA-04 · corregido en V2 §C-06 |

**Tabla `inventario`** — se crea una fila por cada materia prima (R-01)

| Columna | Tipo | Notas |
|---|---|---|
| `id_inventario` | `int unsigned` PK AI | |
| `id_materia_prima` | `int unsigned` NOT NULL **UNIQUE** FK | Una fila por materia prima |
| `cantidad_disponible` | `decimal(10,2)` NOT NULL | `CHECK >= 0` (V2) |
| `fecha_actualizacion` | `datetime` NOT NULL DEFAULT `CURRENT_TIMESTAMP ON UPDATE` | V2 §C-07 |

```json
// POST /api/v1/materias-primas
{ "nombre": "Harina de trigo", "idUnidad": 1,
  "stockMinimo": 50.00, "costoUnitario": 3200.00 }

// 201 Created
{ "idMateriaPrima": 5, "nombre": "Harina de trigo",
  "unidad": { "idUnidad": 1, "nombre": "Kilogramo", "abreviatura": "kg" },
  "stockMinimo": 50.00, "costoUnitario": 3200.00, "activo": true,
  "cantidadDisponible": 0.00 }
```

**Campos ordenables:** `nombre`, `stockMinimo`, `costoUnitario`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `UNIDAD_INEXISTENTE` | 409 | El `idUnidad` no existe en `unidades_medida` |
| `MATERIA_PRIMA_EN_USO` | 409 | Se intenta desactivar una materia prima usada en recetas activas (advertencia) |

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

### El `DEFAULT 1` de `estado` es una corrección, no algo que ya estuviera

El esquema original declaraba `estado tinyint(1) NOT NULL` **sin valor por defecto**, pese
a que CA-04 exige que las nuevas materias primas nazcan activas.

Sin la corrección de V2, un `INSERT` que omita la columna falla. Y como `clientes` y
`proveedores` **sí** tenían el `DEFAULT`, el comportamiento era inconsistente entre tablas
casi idénticas — el tipo de detalle que produce un error desconcertante en la cuarta
pantalla, cuando las tres anteriores funcionaron.
