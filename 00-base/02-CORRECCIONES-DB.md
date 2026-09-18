# 02 — Correcciones al Esquema de Base de Datos

Auditoría del `db.sql` original contra los criterios de aceptación de las 27 historias.
Cada hallazgo indica qué HU se rompe y cómo se corrige.

**Todo lo aquí descrito está implementado en `migraciones/V2__correcciones.sql`.**
Nadie modifica la base con SQL suelto: se agrega una migración.

---

## Resumen

| # | Hallazgo | Severidad | HU afectadas |
|---|---|---|---|
| C-01 | `detalle_pedido` sin `precio_unitario` | **Bloqueante** | HU-17, HU-25 |
| C-02 | No existe stock de producto terminado por producto | **Bloqueante** | HU-17, HU-19, HU-24 |
| C-03 | No hay entrada de materia prima al inventario | **Bloqueante** | HU-10, HU-13, HU-14, HU-23, HU-27 |
| C-04 | Sin trazabilidad de movimientos de stock (kardex) | Alta | HU-10, HU-17, HU-18, HU-23 |
| C-05 | `detalle_receta` sin UNIQUE(receta, materia prima) | Alta | HU-12 |
| C-06 | `estado` sin `DEFAULT 1` en 4 tablas | Media | HU-01, HU-08, HU-11, HU-12 |
| C-07 | Fechas "automáticas" sin `DEFAULT` | Media | HU-16, HU-18, HU-10 |
| C-08 | `consumo_materia_prima.cantidad_real` nullable | Media | HU-22, HU-26 |
| C-09 | División por cero en el cálculo de eficiencia | Media | HU-22, HU-26 |
| C-10 | `lotes` permite estados imposibles | Media | HU-09, HU-19 |
| C-11 | `bitacora.id_usuario` NOT NULL bloquea auditoría del sistema | Media | HU-04 |
| C-12 | Sin restricción de estados válidos | Media | HU-13, HU-16, HU-18, HU-20 |
| C-13 | Sin UNIQUE en líneas de detalle | Media | HU-14, HU-21, HU-22 |
| C-14 | Sin CHECK de cantidades y precios positivos | Media | Transversal |
| C-15 | Faltan índices para los filtros que piden las HU | Baja | HU-02, HU-05, HU-06, HU-15, HU-25 |
| C-16 | Sin datos semilla (roles, unidades, admin inicial) | Bloqueante operativo | HU-01, HU-03, HU-07 |

---

## C-01 · `detalle_pedido` no guarda el precio — **Bloqueante**

**Problema.** La tabla solo tiene `id_pedido`, `id_producto`, `cantidad`. No hay precio.

Dos consecuencias, y la segunda es la grave:

1. No se puede calcular el total de un pedido. HU-25 CA-04 pide mostrar el detalle del
   pedido; un detalle de venta sin importe no sirve.
2. **Se pierde el histórico.** Si el precio se lee de `productos.precio` al consultar,
   un pedido de hace tres meses se re-valoriza con el precio de hoy. Los reportes cambian
   solos cada vez que alguien edita el catálogo.

**Corrección.** `detalle_pedido` gana `precio_unitario DECIMAL(10,2) NOT NULL`, que se
copia de `productos.precio` **en el momento de agregar la línea** y nunca se recalcula.

Mismo razonamiento aplica a `detalle_compra`, que sí tiene `precio_unitario` — correcto
de origen.

---

## C-02 · No existe el stock de producto terminado por producto — **Bloqueante**

**Problema.** HU-17 CA-04 exige "verificar que haya stock suficiente en
`inventario_producto_terminado`". Pero esa tabla está indexada por **lote**
(`id_lote` UNIQUE), no por producto:

```
inventario_producto_terminado(id_inv_producto, id_lote, cantidad_disponible, fecha_ingreso_estante)
lotes(id_lote, codigo_lote, id_producto, id_materia_prima, ...)
```

Para saber cuántos panes hay disponibles hay que agregar todos los lotes de ese producto.
No es una consulta directa, y peor: **la HU no dice de qué lote descontar** cuando hay
varios.

**Corrección.** Dos partes:

1. **Vista `v_stock_producto_terminado`** que agrega por producto. El backend consulta la
   vista para validar disponibilidad.
2. **Política de salida FEFO** (*First Expired, First Out* — primero el que vence antes),
   declarada como regla de negocio. Es lo correcto para alimentos perecederos: si se
   descontara del lote más nuevo, el viejo caduca en el estante.

La vista excluye lotes vencidos: producto caducado no es stock vendible.

---

## C-03 · Nada suma materia prima al inventario — **Bloqueante**

**Problema.** El flujo completo de inventario de materias primas está incompleto:

- HU-23 **descuenta** de `inventario` al registrar consumo de producción. ✅
- HU-10 **consulta** `inventario`. ✅
- HU-27 **alerta** cuando `inventario` está bajo. ✅
- **Nadie suma.** Ni HU-09 (registrar lote de MP) ni HU-13/HU-14 (compras) dicen que
  se incremente `inventario`.

Resultado: la tabla `inventario` arranca vacía, nunca crece, y toda la Fase 3, 7 y 8
opera sobre ceros. Tres historias completas quedan sin sentido.

**Corrección — decisión de negocio.** Se definen **dos puntos de entrada**, ambos
transaccionales y ambos generan movimiento de kardex:

| Punto de entrada | Cuándo | Qué hace |
|---|---|---|
| **Recepción de compra** (principal) | `PATCH /compras/{id}/estado` → `RECIBIDA` | Suma al inventario cada línea de `detalle_compra`. Opcionalmente crea lotes. |
| **Registro de lote de MP** (secundario) | `POST /lotes-materia-prima` (HU-09) | Suma `cantidad` al inventario de esa materia prima. |

**Importante:** si una compra se recibe *y además* se registran lotes por esas mismas
cantidades, el stock se duplica. Regla: **un lote registrado contra una compra recibida
no vuelve a sumar**; se marca con `id_detalle_compra`. Ver `V2__correcciones.sql`.

La transición `RECIBIDA → PENDIENTE` (revertir recepción) se prohíbe: una vez que el
stock entró, deshacerlo requiere un movimiento de ajuste explícito, no un cambio de estado.

---

## C-04 · Sin kardex: el stock muta sin dejar rastro

**Problema.** Cuatro operaciones distintas modifican `inventario.cantidad_disponible`
(recepción de compra, registro de lote, consumo de producción, ajuste) y tres modifican
`inventario_producto_terminado` (alta de lote, venta, devolución aprobada). Pero solo se
guarda el **saldo actual**.

Cuando el contador diga "aquí faltan 40 kg de harina", no habrá forma de saber qué pasó.
La bitácora de HU-04 no sustituye esto: registra "alguien actualizó la tabla inventario",
con un `varchar(255)` de detalle. No es un libro de movimientos consultable.

**Corrección.** Dos tablas de kardex, `movimientos_inventario_mp` y
`movimientos_inventario_pt`, con:

- Tipo de movimiento (`ENTRADA_COMPRA`, `ENTRADA_LOTE`, `SALIDA_PRODUCCION`,
  `SALIDA_VENTA`, `ENTRADA_DEVOLUCION`, `AJUSTE_POSITIVO`, `AJUSTE_NEGATIVO`, `BAJA_VENCIMIENTO`)
- Cantidad, **saldo anterior y saldo posterior** (permite detectar inconsistencias)
- Referencia al documento que lo originó (`tabla_origen` + `id_origen`)
- Usuario y fecha

Regla: **`inventario` nunca se actualiza sin insertar el movimiento correspondiente, en la
misma transacción.** Se centraliza en un único `InventarioService` con métodos
`registrarEntrada()` / `registrarSalida()`; ningún otro servicio hace `UPDATE inventario`.

Esto convierte HU-10 y HU-27 en consultas auditables y da base real a HU-26.

---

## C-05 · `detalle_receta` admite ingredientes duplicados

**Problema.** HU-12 CA-05: *"No se permite duplicar la misma materia prima en una receta"*.
La tabla no tiene esa restricción. Validarlo solo en el servicio deja la puerta abierta a
inserciones concurrentes: dos peticiones simultáneas pasan ambas la comprobación
`SELECT ... WHERE` y ambas insertan.

**Corrección.** `UNIQUE KEY uk_detalle_receta (id_receta, id_materia_prima)`.

El servicio sigue validando para devolver un `409` legible, pero la base es la que
garantiza. Es la diferencia entre un mensaje amable y una regla real.

---

## C-06 · `estado` sin valor por defecto

**Problema.** Cuatro HU dicen "estado por defecto activo (`1`)":

| Tabla | Columna | HU | Estado en `db.sql` |
|---|---|---|---|
| `usuarios` | `estado` | HU-01 CA-05 | `tinyint(1) NOT NULL` sin default |
| `materias_primas` | `estado` | HU-08 CA-04 | `tinyint(1) NOT NULL` sin default |
| `productos` | `estado` | HU-11 CA-05 | `tinyint(1) NOT NULL` sin default |
| `recetas` | `estado` | HU-12 | `tinyint(1) NOT NULL` sin default |

`clientes` y `proveedores` sí lo tienen. La inconsistencia es fuente segura de bugs:
un `INSERT` que omita la columna falla en unas tablas y no en otras.

**Corrección.** `DEFAULT 1` en las cuatro.

---

## C-07 · Fechas "automáticas" que no lo son

| Tabla.columna | La HU dice | `db.sql` dice |
|---|---|---|
| `pedidos.fecha_pedido` | HU-16 CA-02: "se registra automáticamente con la fecha actual" | `date NOT NULL` sin default |
| `devoluciones.fecha_devolucion` | HU-18 CA-05: "se registra automáticamente" | `date NOT NULL` sin default |
| `inventario.fecha_actualizacion` | HU-10 CA-02 / HU-23 CA-02 | `datetime NOT NULL` sin default |

**Corrección.** `DEFAULT curdate()` en las dos primeras;
`DEFAULT current_timestamp() ON UPDATE current_timestamp()` en la tercera — así se
actualiza sola en cada `UPDATE` y se elimina la posibilidad de que alguien olvide tocarla.

---

## C-08 · `cantidad_real` nullable contra su propio criterio

**Problema.** HU-22 CA-04: *"`cantidad_real` es obligatoria (lo que realmente se usó)"*.
La columna es `decimal(10,3) DEFAULT NULL`.

Además rompe HU-23 CA-01, que descuenta `cantidad_real` del inventario: descontar `NULL`
propaga `NULL` a `cantidad_disponible` y corrompe el saldo.

**Corrección.** `NOT NULL`. Si el flujo real necesita registrar el consumo teórico antes
de conocer el real, eso es un estado distinto del detalle del plan, no un `NULL` en una
columna de cantidad.

---

## C-09 · La fórmula de eficiencia divide por cero

**Problema.** HU-22 CA-06 define `eficiencia = (cantidad_teorica / cantidad_real) * 100`.

Si `cantidad_real = 0` — perfectamente posible: se planificó un producto y no se consumió
nada de ese ingrediente — la división revienta. MySQL devuelve `NULL` en lugar de lanzar
error, así que **falla en silencio** y el reporte de HU-26 muestra huecos sin explicación
aparente.

Hay un segundo problema, conceptual: con esa fórmula, consumir *menos* de lo teórico da
más de 100%. Un 200% de eficiencia no significa el doble de bueno, significa que se usó
la mitad del ingrediente — lo cual probablemente sea un error de registro, no una hazaña.

**Corrección.**
- En SQL: `CASE WHEN cantidad_real = 0 THEN NULL ELSE (cantidad_teorica / cantidad_real) * 100 END`.
- En la capa de servicio: se calcula con `BigDecimal`, se devuelve `null` cuando
  `cantidad_real = 0`, y el frontend muestra `—` en vez de `NaN` o `Infinity`.
- Se documenta que **la desviación es la métrica interpretable**:
  `desviacion% = ((real - teorica) / teorica) * 100`. Positiva = se gastó de más.
  HU-26 muestra ambas.

---

## C-10 · `lotes` acepta estados imposibles

**Problema.** `id_producto` e `id_materia_prima` son ambos nullable y no hay restricción.
La tabla permite:

- Un lote con los dos nulos (no es de nada).
- Un lote con los dos llenos (¿es harina o es pan?).
- `fecha_vencimiento` anterior a `fecha_produccion`.

**Corrección.** Dos CHECK (MySQL los aplica desde 8.0.16; el proyecto usa 8.4):

```sql
CHECK ((id_producto IS NOT NULL AND id_materia_prima IS NULL)
    OR (id_producto IS NULL AND id_materia_prima IS NOT NULL))
CHECK (fecha_vencimiento >= fecha_produccion)
```

---

## C-11 · La bitácora no puede registrar acciones del sistema

**Problema.** `bitacora.id_usuario` es `NOT NULL` con FK a `usuarios`. Pero HU-04 CA-01
dice "se registra automáticamente al crear, editar o eliminar". Hay acciones sin usuario
autenticado:

- La creación del primer administrador (semilla).
- Intentos de login fallidos (HU-03 CA-04) — no hay usuario válido que registrar.
- Procesos programados (baja de lotes vencidos).

**Corrección.** `id_usuario` pasa a nullable. `NULL` significa "sistema", y se documenta
así. El frontend muestra "Sistema" cuando viene nulo.

---

## C-12 · Estados libres: `varchar(30)` sin restricción

**Problema.** Cuatro tablas guardan el estado como texto libre. Las HU dan ejemplos
("Pendiente", "Recibida", "Cancelada") pero nada impide `"pendiente"`, `"PENDIENTE"`,
`"Pendinte"`. Cada filtro por estado se vuelve una lotería, y HU-15 CA-03, HU-24 CA-02 y
HU-25 CA-03 filtran exactamente por ahí.

**Corrección.** Valores canónicos en `SCREAMING_SNAKE`, con CHECK en la base y `enum` en
Java y TypeScript:

| Tabla | Valores permitidos |
|---|---|
| `compras.estado` | `PENDIENTE`, `RECIBIDA`, `CANCELADA` |
| `pedidos.estado` | `PENDIENTE`, `EN_PREPARACION`, `ENTREGADO`, `CANCELADO` |
| `devoluciones.estado` | `PENDIENTE`, `APROBADA`, `RECHAZADA` |
| `planes_produccion.estado` | `PLANIFICADO`, `EN_PROCESO`, `COMPLETADO`, `CANCELADO` |

La etiqueta legible ("En preparación") se resuelve en el frontend. La base guarda el código.

Las **transiciones válidas** están en `04-CONTRATO-API.md §7`.

---

## C-13 · Líneas de detalle duplicables

Además de C-05, faltan restricciones equivalentes en:

| Tabla | UNIQUE necesario | Por qué |
|---|---|---|
| `detalle_compra` | `(id_compra, id_materia_prima)` | Dos líneas de la misma MP en una compra: el total es ambiguo y la recepción suma doble |
| `detalle_pedido` | `(id_pedido, id_producto)` | Idem para HU-17 |
| `detalle_plan_produccion` | `(id_plan, id_producto)` | HU-21: dos cantidades planificadas para el mismo producto |
| `consumo_materia_prima` | `(id_detalle_plan, id_materia_prima)` | HU-22/HU-23: descontaría el inventario dos veces |

En todos los casos, agregar la misma línea otra vez debe **sumar a la existente** o
devolver `409`, decisión que se documenta por HU. Nunca crear un duplicado silencioso.

---

## C-14 · Sin CHECK de valores positivos

Ninguna columna de cantidad, precio o stock impide negativos. Un `precio = -5000` o una
`cantidad = -10` pasan sin protesta, y esta última corrompe el inventario al descontarse.

**Corrección.** CHECK en las columnas de dominio: cantidades `> 0` en líneas de detalle,
saldos `>= 0` en inventarios, precios y costos `>= 0`, `stock_minimo >= 0`.

El `>= 0` en `inventario.cantidad_disponible` es la red de seguridad contra el bug clásico
de vender más de lo que hay: aunque la validación del servicio falle, la base rechaza.

---

## C-15 · Faltan índices para los filtros que piden las HU

Las HU piden explícitamente filtrar por campos sin índice. Con pocos registros no se nota;
con datos reales, cada listado es un *full table scan*.

| Índice | Lo pide |
|---|---|
| `clientes(nombre)` | HU-05 CA-04 |
| `proveedores(nombre)` | HU-06 CA-04 |
| `materias_primas(nombre)` | HU-08 CA-05 |
| `productos(nombre)` | HU-11 CA-06 |
| `usuarios(nombre)` | HU-02 CA-02 |
| `compras(fecha)` | HU-15 CA-02 |
| `pedidos(fecha_pedido)`, `pedidos(estado)` | HU-25 CA-02, CA-03 |
| `planes_produccion(fecha_produccion)` | HU-24 CA-04, HU-26 CA-04 |
| `lotes(fecha_vencimiento)` | HU-24 CA-05 |
| `bitacora(tabla_afectada, fecha)` | HU-04 |

Nota: un índice B-tree **no** acelera `LIKE '%texto%'` (comodín inicial). Sí acelera
`LIKE 'texto%'`. Los filtros por nombre se implementan como **prefijo** por defecto
(`texto%`), que además es lo que el usuario espera al escribir en un buscador. Se
documenta en `04-CONTRATO-API.md §4`.

---

## C-16 · Sin datos semilla

El esquema no trae ni un registro. Sin esto el sistema es inarrancable:

- **`roles`**: HU-01 CA-04 exige que el rol exista. Sin filas, no se puede crear ningún usuario.
- **`usuarios`**: sin un administrador inicial nadie puede entrar a crear el primero.
  Problema del huevo y la gallina.
- **`unidades_medida`**: HU-08 CA-02 y HU-11 CA-04 exigen que la unidad exista.

**Corrección.** `V3__datos_semilla.sql` con los 5 roles, un catálogo básico de unidades y
un administrador inicial.

> ⚠️ **El administrador semilla tiene contraseña conocida y debe cambiarse antes de
> cualquier despliegue fuera de desarrollo local.** Está documentado en
> `09-SETUP-ENTORNO.md` y marcado como tarea de cierre en HU-01.

---

## Notas de MySQL 8.4 y AWS RDS

El motor es **MySQL 8.4 LTS en Amazon RDS** (ver `00-STACK-VERSIONES.md`). Puntos que
afectan directamente a estas correcciones:

### Sintaxis que difiere de MariaDB

La tabla de stack de las historias mencionaba MariaDB, y es probable que alguien copie SQL
de un tutorial de MariaDB. Esto es lo que va a fallar:

| Construcción | MySQL 8.4 | Consecuencia si se escribe como MariaDB |
|---|---|---|
| Expresión en `DEFAULT` | `DEFAULT (curdate())` | Error de sintaxis. Falla al ejecutar, fácil de ver |
| Ancho de entero | `int unsigned` | `int(10)` funciona pero avisa obsolescencia |
| `ORDER BY` en una vista | **Se ignora** | ⚠️ La vista se crea sin protestar y devuelve orden arbitrario |

La tercera es la peligrosa. Por eso `v_lotes_disponibles_fefo` **no** confía en su propio
`ORDER BY`: el orden FEFO se aplica en la consulta que la consume. Un FEFO que silenciosamente
deja de ordenar por vencimiento hace que el producto viejo caduque en el estante mientras se
vende el nuevo — y nada en el sistema lo señala.

### Collation

Se adopta `utf8mb4_0900_ai_ci`, la nativa de MySQL 8. Es **accent-insensitive**, lo que
significa que buscar `"panaderia"` encuentra `"Panadería"`. Para los filtros por nombre de
HU-05, HU-06, HU-08 y HU-11 es exactamente el comportamiento que el usuario espera.

Efecto secundario a tener presente: también aplica a los `UNIQUE`. Dos unidades de medida
llamadas `"Kilogramo"` y `"kilográmo"` se consideran **la misma** y la segunda es rechazada.
En este dominio es correcto.

### Restricciones propias de RDS

| Limitación | Impacto en este esquema |
|---|---|
| Sin privilegio `SUPER` | No se usan triggers ni funciones almacenadas. Las reglas viven en el servicio. Ninguna corrección de este documento las necesita |
| Variables solo vía *Parameter Group* | `time_zone` se configura ahí — ver abajo |
| `DEFINER` en vistas | Las tres vistas se crean con `SQL SECURITY INVOKER` para que sobrevivan a la restauración de una snapshot con otro usuario |
| `sql_mode` estricto por defecto | Un `INSERT` con datos que no caben **falla** en vez de truncar. Es lo correcto, pero rompe código escrito asumiendo MySQL permisivo |

### Zona horaria — el error que va a aparecer

**Una instancia RDS nueva está en UTC.** Colombia es UTC-5.

Consecuencia concreta: un pedido registrado a las 7:30 p.m. del 18 de septiembre se guarda
con fecha **19 de septiembre**. HU-24 CA-01 ("total de pedidos del día actual") y HU-25
dejan de cuadrar con lo que el usuario ve en pantalla, y el error solo se manifiesta después
de las 7 p.m. — lo que hace que en las pruebas de la mañana todo parezca correcto.

Dos medidas, ambas necesarias:

1. En el *Parameter Group* de RDS: `time_zone = America/Bogota`.
2. En la aplicación: `spring.jackson.time-zone=America/Bogota` y arrancar la JVM con
   `-Duser.timezone=America/Bogota`.

La columna `fecha_pedido` es `DATE` con `DEFAULT (curdate())`, y `curdate()` la evalúa **el
servidor de base de datos**. Sin el punto 1, el punto 2 no salva la situación.
