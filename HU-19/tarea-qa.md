# HU-19 · Lotes de producto terminado — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..04 | Registrar lote válido | Código, producto, fechas, 25 unidades | `201` con `cantidadDisponible: 25.00` | Positivo |
| CP-02 | CA-05 | 🔴 **Se crea la fila de inventario** | Tras CP-01 | Existe fila en `inventario_producto_terminado` con 25.00 (R-02) | Integridad |
| CP-03 | CA-05 | 🔴 **Se registra el movimiento de entrada** | Tras CP-01 | `ENTRADA_PRODUCCION` en `movimientos_inventario_pt` | Integridad |
| CP-04 | CA-05 | 🔴 **El producto pasa a tener stock vendible** | Tras CP-01, consultar `GET /productos/3/stock` | El disponible incluye las 25 unidades (R-01) | Integridad |
| CP-05 | CA-01 | Código duplicado | Repetir `codigoLote` | `409 CODIGO_LOTE_DUPLICADO` | Negativo |
| CP-06 | CA-01 | 🔴 **Código que ya usa un lote de materia prima** | Usar un código de HU-09 | `409` — la tabla `lotes` es compartida | Borde |
| CP-07 | CA-01 | Código con otra capitalización | `"pf-2026-0918"` existiendo el mayúsculas | `409` — se normaliza (R-08) | Borde |
| CP-08 | CA-02 | Producto inexistente | `{"idProducto":9999}` | `409 PRODUCTO_INEXISTENTE` | Negativo |
| CP-09 | CA-03 | Vencimiento anterior a producción | `venc < prod` | `400 FECHAS_INVALIDAS` | Negativo |
| CP-10 | CA-03 | 🔴 **Lote ya vencido** | `fechaVencimiento` de ayer | `400 LOTE_YA_VENCIDO` (R-05) | Negativo |
| CP-11 | CA-03 | Vence hoy | `fechaVencimiento` = hoy | `201` — todavía es vendible | Borde |
| CP-12 | CA-04 | Cantidad cero | `{"cantidad":0}` | `400 CANTIDAD_INVALIDA` | Negativo |
| CP-13 | CA-04 | Cantidad negativa | `{"cantidad":-5}` | `400` | Negativo |
| CP-14 | CA-06 | Filtrar por producto | `?idProducto=3` | Solo lotes de ese producto | Positivo |
| CP-15 | CA-06 | Filtrar próximos a vencer | `?proximosAVencer=7` | Solo los que vencen en ≤ 7 días | Positivo |
| CP-16 | CA-06 | Filtrar solo vencidos | `?soloVencidos=true` | Solo los vencidos | Positivo |
| CP-17 | — | 🔴 **`lotes.cantidad` no cambia al vender** | Vender 10 de un lote de 25 | `lotes.cantidad` sigue en 25; el disponible baja a 15 (R-03) | Integridad |
| CP-18 | — | 🔴 **Un lote vencido no cuenta como stock** | Lote con 20 unidades vencido ayer | `v_stock_producto_terminado` lo excluye (R-04) | Integridad |
| CP-19 | — | El lote vencido sigue existiendo | Tras CP-18 | La fila y su saldo permanecen; solo deja de ser vendible | Borde |
| CP-20 | — | 🔴 **Exclusividad producto/materia prima** | `INSERT` directo con ambos informados | La base lo rechaza — `ck_lotes_exclusividad` (R-06) | Integridad |
| CP-21 | — | Transaccionalidad | Provocar fallo en el `INSERT` de inventario | **No se crea el lote tampoco** (R-02) | Integridad |
| CP-22 | — | Kardex del lote | `GET /lotes-producto/31/movimientos` | Entrada, ventas y devoluciones del lote | Positivo |
| CP-23 | — | Rol VENTAS no puede registrar lotes | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-24 | — | Rol PRODUCCION puede registrar | Token de PRODUCCION | `201` | Positivo |

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

> **CP-02 y CP-17 son los casos críticos.**
>
> **CP-02:** un lote sin fila de inventario se registra sin dar ningún error, aparece en el
> listado de lotes y es invisible para las ventas. El síntoma llega después, como "registré
> el lote pero no puedo vender el producto".
>
> **CP-17:** confundir `lotes.cantidad` con el saldo disponible es el error natural al leer
> el esquema. Si alguna ruta de código actualiza `lotes.cantidad` al vender, se pierde el
> dato de cuánto ingresó el lote y la trazabilidad deja de cuadrar.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-02 y CP-21: todo lote de producto DEBE tener su fila de inventario
SELECT l.id_lote, l.codigo_lote, l.cantidad AS ingresada, ipt.cantidad_disponible
FROM lotes l
LEFT JOIN inventario_producto_terminado ipt USING (id_lote)
WHERE l.id_producto IS NOT NULL AND ipt.id_inv_producto IS NULL;
-- DEBE devolver 0 filas. Un lote sin inventario es invisible para las ventas
-- y no produce ningún error: parece que se registró bien.

-- 🔴 CP-17: la cantidad del lote NO es el saldo
SELECT l.codigo_lote, l.cantidad AS ingreso_original, ipt.cantidad_disponible AS saldo_actual,
       l.cantidad - ipt.cantidad_disponible AS vendido
FROM lotes l JOIN inventario_producto_terminado ipt USING (id_lote)
WHERE l.id_producto = 3;
-- ingreso_original debe permanecer constante; saldo_actual baja con cada venta.
-- Si ambos bajan juntos, se está actualizando lotes.cantidad: defecto contra R-03.

-- 🔴 CP-18: la vista excluye vencidos
SELECT * FROM v_stock_producto_terminado WHERE id_producto = 3;
SELECT l.codigo_lote, l.fecha_vencimiento, ipt.cantidad_disponible
FROM lotes l JOIN inventario_producto_terminado ipt USING (id_lote)
WHERE l.id_producto = 3 ORDER BY l.fecha_vencimiento;
-- El total de la vista NO debe incluir los lotes con fecha_vencimiento < hoy

-- CP-20: el CHECK de exclusividad de V2 §C-10
INSERT INTO lotes (codigo_lote, id_producto, id_materia_prima,
                   fecha_produccion, fecha_vencimiento, cantidad)
VALUES ('X-99', 3, 5, curdate(), curdate(), 10);
-- DEBE fallar con error 3819

-- Lotes vencidos que todavía tienen stock: hay que retirarlos físicamente
SELECT l.codigo_lote, p.nombre, l.fecha_vencimiento, ipt.cantidad_disponible
FROM lotes l
  JOIN productos p USING (id_producto)
  JOIN inventario_producto_terminado ipt USING (id_lote)
WHERE l.fecha_vencimiento < curdate() AND ipt.cantidad_disponible > 0;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
