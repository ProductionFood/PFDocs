# HU-12 · Gestión de recetas — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..03 | Crear receta con detalle | Cabecera + 3 ingredientes | `201` con receta y detalles | Positivo |
| CP-02 | CA-01 | 🔴 **Segunda receta para el mismo producto** | `POST` con un `idProducto` que ya tiene receta | `409 PRODUCTO_YA_TIENE_RECETA` | Negativo |
| CP-03 | CA-02 | Nombre vacío | `{"nombre":""}` | `400` | Negativo |
| CP-04 | CA-02 | `cantidadProducir` en cero | `{"cantidadProducir":0}` | `400 CANTIDAD_PRODUCIR_INVALIDA` (R-03) | Negativo |
| CP-05 | CA-02 | `cantidadProducir` negativa | `{"cantidadProducir":-5}` | `400` | Negativo |
| CP-06 | CA-02 | Unidad inexistente | `{"idUnidad":9999}` | `409` | Negativo |
| CP-07 | CA-02 | Estado por defecto | Crear sin `estado` | `activo: true` | Positivo |
| CP-08 | CA-03 | Listar el detalle | `GET /recetas/2` | Detalles con nombre y unidad de cada materia prima | Positivo |
| CP-09 | CA-04 | Agregar un ingrediente | `POST /recetas/2/detalles` | `201`; el detalle crece | Positivo |
| CP-10 | CA-04 | Editar la cantidad | `PUT /recetas/2/detalles/11 {"cantidad":6.5}` | `200`; costo recalculado | Positivo |
| CP-11 | CA-04 | Quitar un ingrediente | `DELETE /recetas/2/detalles/11` | `204`; el detalle decrece (R-07) | Positivo |
| CP-12 | CA-05 | 🔴 **Ingrediente duplicado** | `POST` con una materia prima ya presente | `409 MATERIA_PRIMA_DUPLICADA` | Negativo |
| CP-13 | CA-05 | 🔴 **Duplicado dentro del POST inicial** | Crear receta con la misma MP dos veces en `detalles[]` | `409`; **no se crea la receta** | Negativo |
| CP-14 | CA-05 | 🔴 **Duplicado por SQL directo** | `INSERT` directo en `detalle_receta` repitiendo la MP | La base lo rechaza — `uk_detalle_receta` (R-01) | Integridad |
| CP-15 | — | Cantidad con 3 decimales | `{"cantidad":0.005}` | `201`; en base `0.005` exacto (R-06) | Borde |
| CP-16 | — | Cantidad cero en el detalle | `{"cantidad":0}` | `400` — `CHECK > 0` | Negativo |
| CP-17 | — | Producto inexistente | `{"idProducto":9999}` | `409 PRODUCTO_INEXISTENTE` | Negativo |
| CP-18 | — | Materia prima inexistente | `{"idMateriaPrima":9999}` | `409 MATERIA_PRIMA_INEXISTENTE` | Negativo |
| CP-19 | — | Receta sin ingredientes | Crear solo la cabecera | `201`; marcada como incompleta (R-05) | Borde |
| CP-20 | — | Cálculo del costo estimado | 5 kg × $3.200 + 0,05 kg × $180 | `costoTotalEstimado` = `16009.00` | Positivo |
| CP-21 | — | Costo por unidad | `costoTotal` 16.009 / `cantidadProducir` 30 | `533.63` (redondeo HALF_UP) | Positivo |
| CP-22 | — | Transacción al crear | Provocar fallo en el tercer detalle | **No se crea ni la receta ni los dos primeros detalles** | Integridad |
| CP-23 | — | Rol CONSULTA no puede editar | `POST` con token CONSULTA | `403 SIN_PERMISO` | Seguridad |
| CP-24 | — | Rol PRODUCCION puede editar | `POST` con token PRODUCCION | `201` | Positivo |

---

## 2. Batería de seguridad

Obligatoria en **cada endpoint** de esta historia:

| ID | Caso | Esperado |
|---|---|---|
| SEC-01 | Sin cabecera `Authorization` | `401 NO_AUTENTICADO` |
| SEC-02 | Token malformado | `401 NO_AUTENTICADO` |
| SEC-03 | Token expirado | `401 NO_AUTENTICADO` |
| SEC-04 | Token de rol **sin** permiso | `403 SIN_PERMISO` |
| SEC-05 | Token de usuario con `estado = 0` | `403 USUARIO_INACTIVO` |

> **SEC-04 es el caso crítico.** Si devuelve `200`, falta `@EnableMethodSecurity` y la
> autorización del sistema completo está desactivada.

> **CP-14 y la consulta final son las pruebas que evitan un desastre tardío.**
>
> Un ingrediente duplicado hace que HU-22 calcule el doble de consumo teórico y HU-23
> descuente el doble de inventario. Una `cantidad_producir` mal interpretada multiplica
> todos los cálculos de producción por el rendimiento de la receta.
>
> Ninguno de los dos errores produce un mensaje. Ambos se manifiestan en la Fase 7, lejos
> de su causa, como "el inventario no cuadra" y "la eficiencia da números raros".

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-14: la restricción de V2 §C-05. Esta sentencia DEBE fallar.
INSERT INTO detalle_receta (id_receta, id_materia_prima, cantidad) VALUES (2, 5, 1.000);
INSERT INTO detalle_receta (id_receta, id_materia_prima, cantidad) VALUES (2, 5, 2.000);
-- La segunda debe dar error 1062 (Duplicate entry).
-- Si ambas pasan, falta el UNIQUE y HU-23 descontará el doble de inventario.

SHOW CREATE TABLE detalle_receta\G
-- Se espera: UNIQUE KEY `uk_detalle_receta` (`id_receta`,`id_materia_prima`)

-- CP-02: un producto, una receta
SHOW CREATE TABLE recetas\G
-- Se espera: UNIQUE KEY `id_producto` (`id_producto`)

-- CP-15: los 3 decimales se conservan
SELECT id_detalle_receta, cantidad FROM detalle_receta WHERE id_receta = 2;
-- 0.005 debe verse como 0.005, no 0.01

-- CP-22: tras el fallo, ni receta ni detalles
SELECT COUNT(*) FROM recetas WHERE id_producto = 3;
SELECT COUNT(*) FROM detalle_receta WHERE id_receta = <la que no debió crearse>;

-- Verificación de R-02: el consumo teórico que usará HU-22
SELECT r.nombre, r.cantidad_producir, mp.nombre AS ingrediente, dr.cantidad,
       ROUND(dr.cantidad / r.cantidad_producir, 4) AS por_unidad
FROM recetas r
  JOIN detalle_receta dr USING (id_receta)
  JOIN materias_primas mp USING (id_materia_prima)
WHERE r.id_receta = 2;
-- "por_unidad" debe tener sentido físico: 0.1667 kg de harina por pan es razonable;
-- 5 kg de harina por pan indica que cantidad_producir se interpretó mal.
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
