# HU-09 · Registro de lotes de materia prima — Especificación

**Fase 3** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** registrar lotes de materia prima con código, fechas de producción/vencimiento y cantidad,
> **para que** controle la vida útil de los ingredientes.

**Depende de:** [HU-08](../HU-08/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `codigo_lote` obligatorio y **único**, máx. 30 | `UNIQUE KEY codigo_lote` ya existe; validación previa en el servicio |
| CA-02 | `id_materia_prima` debe existir | Validado contra `MateriaPrimaRepository` |
| CA-03 | `fecha_produccion` y `fecha_vencimiento` obligatorias | `@NotNull`; `CHECK (vencimiento >= produccion)` añadido en V2 §C-10 |
| CA-04 | `cantidad` obligatoria (decimal) | `@NotNull` + `@DecimalMin` positivo; `CHECK > 0` en V2 |
| CA-05 | Listar lotes por materia prima | `GET /lotes-materia-prima?idMateriaPrima=` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/lotes-materia-prima` | Listar con filtros y paginación | ADMIN, PRODUCCION, COMPRAS, CONSULTA |
| `GET` | `/api/v1/lotes-materia-prima/{id}` | Obtener uno | ADMIN, PRODUCCION, COMPRAS, CONSULTA |
| `POST` | `/api/v1/lotes-materia-prima` | Registrar un lote | ADMIN, PRODUCCION, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Registrar un lote **suma al inventario**

Es uno de los dos puntos de entrada de stock del sistema (`02-CORRECCIONES-DB.md` §C-03).
La operación es **transaccional** y hace tres cosas:

1. `INSERT` en `lotes`;
2. `UPDATE inventario` sumando la cantidad;
3. `INSERT` en `movimientos_inventario_mp` con tipo `ENTRADA_LOTE`.

Las tres o ninguna. Toda la suma pasa por `InventarioService.registrarEntrada()`; ningún
otro componente ejecuta `UPDATE inventario`.

### R-02 · ⚠️ El riesgo de la doble suma

El otro punto de entrada es la recepción de compra (HU-13). **Si se recibe una compra de
100 kg de harina y después se registra un lote por esos mismos 100 kg, el inventario
marcará 200 kg.** El error no produce ningún mensaje: simplemente el sistema cree que hay
el doble de harina de la que hay.

Mitigación, en dos partes:

- V2 añadió `lotes.id_detalle_compra` (§C-03). **Un lote creado desde la recepción de una
  compra lleva esa referencia y NO vuelve a sumar**: el stock ya entró con la recepción.
- Un lote registrado manualmente (sin `idDetalleCompra`) **sí suma**. Es la vía para
  inventario inicial o ajustes de entrada.

La interfaz debe dejar claro cuál de los dos casos se está usando. Es el punto donde más
fácilmente se corrompe el inventario, y el más difícil de detectar después.

### R-03 · Un lote es de materia prima **o** de producto terminado, nunca ambos
La tabla `lotes` es compartida con HU-19. El `CHECK ck_lotes_exclusividad` de V2 (§C-10)
impide los estados imposibles que el esquema original permitía: ambos nulos o ambos llenos.

En esta historia `id_materia_prima` va informado e `id_producto` va `NULL`.

### R-04 · La fecha de vencimiento no puede ser anterior a la de producción
`CHECK (fecha_vencimiento >= fecha_produccion)` en V2. Se permite que sean iguales: hay
insumos de un solo día.

### R-05 · Se permite registrar lotes ya vencidos
No se bloquea: puede ser necesario registrar inventario existente que ya caducó, para darlo
de baja formalmente. **Pero se marca visualmente** y HU-24 CA-05 los reporta.

Un lote vencido de materia prima **sí suma al inventario**, porque físicamente está en la
bodega. Darlo de baja es una decisión explícita, no automática.

### R-06 · El código de lote se normaliza
Se recorta y se guarda en mayúsculas. La collation es *case-insensitive*, así que
`h-2026-001` y `H-2026-001` colisionarían de todos modos; normalizar evita que la lista se
vea desordenada.

---

## 4. Modelo de datos

**Tabla `lotes`** (compartida con HU-19; V2 añade `id_detalle_compra` y dos CHECK)

| Columna | Tipo | Notas |
|---|---|---|
| `id_lote` | `int unsigned` PK AI | |
| `codigo_lote` | `varchar(30)` NOT NULL **UNIQUE** | CA-01 |
| `id_producto` | `int unsigned` NULL | **`NULL` en esta historia** (R-03) |
| `id_materia_prima` | `int unsigned` NULL | Informado aquí |
| `id_detalle_compra` | `int unsigned` NULL | **Añadido en V2** — evita la doble suma (R-02) |
| `fecha_produccion` | `date` NOT NULL | CA-03 |
| `fecha_vencimiento` | `date` NOT NULL | CA-03 · `CHECK >= produccion` |
| `cantidad` | `decimal(10,2)` NOT NULL | CA-04 · `CHECK > 0` |

**Tabla `movimientos_inventario_mp`** (creada en V2 §C-04) — recibe el movimiento
`ENTRADA_LOTE`.

```json
// POST /api/v1/lotes-materia-prima
{ "codigoLote": "H-2026-0912", "idMateriaPrima": 5,
  "fechaProduccion": "2026-09-10", "fechaVencimiento": "2027-03-10",
  "cantidad": 100.00 }

// 201 Created
{ "idLote": 14, "codigoLote": "H-2026-0912",
  "materiaPrima": { "idMateriaPrima": 5, "nombre": "Harina de trigo", "unidad": "kg" },
  "fechaProduccion": "2026-09-10", "fechaVencimiento": "2027-03-10",
  "cantidad": 100.00, "diasParaVencer": 173, "vencido": false,
  "inventarioActualizado": { "anterior": 12.50, "posterior": 112.50 } }
```

Devolver el saldo anterior y posterior hace visible el efecto sobre el inventario en el
momento, en lugar de obligar a ir a otra pantalla a comprobarlo.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `CODIGO_LOTE_DUPLICADO` | 409 | Ya existe un lote con ese código |
| `MATERIA_PRIMA_INEXISTENTE` | 409 | El `idMateriaPrima` no existe |
| `FECHAS_INVALIDAS` | 400 | `fechaVencimiento` anterior a `fechaProduccion` |
| `CANTIDAD_INVALIDA` | 400 | `cantidad` menor o igual a cero |

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

### Sobre `cantidad`: el lote no lleva saldo propio

`lotes.cantidad` es la cantidad **con la que el lote ingresó**, y no se modifica después.
El saldo disponible de materia prima vive en `inventario`, agregado por materia prima.

Esto significa que **no hay trazabilidad por lote en el consumo** de materias primas: al
producir (HU-22) se descuenta del total, no de un lote concreto. Para producto terminado sí
la hay, porque `inventario_producto_terminado` es por lote.

Es una limitación del esquema original, no un descuido de esta historia. Se registra por si
la trazabilidad completa se vuelve un requisito: implicaría llevar saldo por lote de materia
prima y consumir en orden FEFO, como ya se hace con el producto terminado.
