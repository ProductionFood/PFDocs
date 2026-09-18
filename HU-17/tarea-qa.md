# HU-17 · Agregar productos al pedido — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..04 | Agregar producto con stock suficiente | 30 unidades, hay 77 | `201` con `lotesConsumidos` | Positivo |
| CP-02 | CA-04 | 🔴 **El stock se descuenta** | Tras CP-01 | Disponible baja de 77 a 47 | Integridad |
| CP-03 | CA-04 | 🔴 **FEFO: se consume primero el que vence antes** | Lotes: 12 und (19/09), 40 und (21/09); pedir 30 | Se toman **12 del primero** y 18 del segundo (R-02) | Integridad |
| CP-04 | CA-04 | 🔴 **FEFO con orden de inserción inverso** | Crear primero el lote que vence después | Igual: manda el vencimiento, no el orden de alta | Integridad |
| CP-05 | CA-04 | Stock insuficiente | Pedir 100, hay 77 | `409 STOCK_INSUFICIENTE` con las cifras en el mensaje | Negativo |
| CP-06 | CA-04 | Stock exacto | Pedir 77, hay 77 | `201`; disponible queda en 0 | Borde |
| CP-07 | CA-04 | Una unidad más que el stock | Pedir 78, hay 77 | `409 STOCK_INSUFICIENTE` | Borde |
| CP-08 | CA-04 | 🔴 **Los lotes vencidos no cuentan** | 100 und vencidas, 0 vigentes | `409 STOCK_INSUFICIENTE` (R-03) | Integridad |
| CP-09 | CA-04 | 🔴 **Concurrencia por el último ítem** | 2 peticiones simultáneas, 1 unidad disponible | Una `201`, otra `409`. **Nunca stock negativo** (R-05) | Integridad |
| CP-10 | CA-02 | Producto inexistente | `{"idProducto":9999}` | `409 PRODUCTO_INEXISTENTE` | Negativo |
| CP-11 | CA-02 | Producto desactivado | Producto con `estado = 0` | `409 PRODUCTO_INACTIVO` | Negativo |
| CP-12 | CA-03 | Cantidad cero | `{"cantidad":0}` | `400 CANTIDAD_INVALIDA` | Negativo |
| CP-13 | CA-03 | Cantidad negativa | `{"cantidad":-5}` | `400` | Negativo |
| CP-14 | — | 🔴 **El precio se copia, no se referencia** | Agregar a $1.200; subir el catálogo a $1.500; consultar | El detalle sigue en `1200.00` (R-04) | Integridad |
| CP-15 | — | Producto duplicado | Agregar uno ya presente | `409 PRODUCTO_DUPLICADO` (R-06) | Negativo |
| CP-16 | — | 🔴 **Aumentar cantidad descuenta la diferencia** | De 10 a 15 | Se descuentan **5**, no 15 (R-07) | Integridad |
| CP-17 | — | 🔴 **Reducir cantidad devuelve la diferencia** | De 10 a 6 | Se devuelven **4** a los lotes de origen | Integridad |
| CP-18 | — | Aumentar sin stock | De 10 a 200 sin disponibilidad | `409 STOCK_INSUFICIENTE`; la línea **no cambia** | Negativo |
| CP-19 | — | 🔴 **Quitar una línea devuelve todo el stock** | `DELETE` de una línea de 30 | Los 30 vuelven a los lotes de origen (R-08) | Integridad |
| CP-20 | — | Agregar a pedido ENTREGADO | `POST` sobre uno entregado | `409 PEDIDO_NO_EDITABLE` (R-09) | Negativo |
| CP-21 | — | Agregar a pedido CANCELADO | `POST` sobre uno cancelado | `409 PEDIDO_NO_EDITABLE` | Negativo |
| CP-22 | — | 🔴 **Transaccionalidad** | Provocar fallo tras descontar el primer lote | **Ningún** lote queda descontado | Integridad |
| CP-23 | — | Trazabilidad lote → pedido | Consultar el kardex tras CP-01 | Movimientos `SALIDA_VENTA` con `tabla_origen = detalle_pedido` (R-10) | Integridad |
| CP-24 | — | Rol COMPRAS no puede agregar | Token de COMPRAS | `403 SIN_PERMISO` | Seguridad |

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

> **CP-03, CP-04 y CP-09 son los casos críticos del proyecto entero.**
>
> **CP-03/CP-04 (FEFO):** si se confía en el `ORDER BY` declarado dentro de la vista, MySQL
> lo ignora y el orden es arbitrario. Todo funciona, las cifras cuadran, y el producto viejo
> caduca en el estante mientras se vende el nuevo. No hay ningún mensaje de error.
>
> **CP-09 (concurrencia):** requiere lanzar dos peticiones en paralelo, por ejemplo con
> `ab -n 2 -c 2` o el *runner* de Postman con dos hilos. Es la única forma de detectar la
> falta de `FOR UPDATE`, y el síntoma en producción es un inventario negativo que nadie
> sabe explicar.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-03 y CP-04: verificar que FEFO se aplicó
SELECT l.codigo_lote, l.fecha_vencimiento, ipt.cantidad_disponible
FROM lotes l JOIN inventario_producto_terminado ipt USING (id_lote)
WHERE l.id_producto = 3 ORDER BY l.fecha_vencimiento;
-- El lote que vence ANTES debe ser el que quedó en cero o más reducido.
-- Si se consumió el más nuevo, FEFO no se está aplicando: probablemente
-- se confió en el ORDER BY de la vista, que MySQL ignora (R-02).

SELECT m.tipo_movimiento, l.codigo_lote, l.fecha_vencimiento,
       m.cantidad, m.saldo_anterior, m.saldo_posterior
FROM movimientos_inventario_pt m JOIN lotes l USING (id_lote)
WHERE m.tabla_origen = 'detalle_pedido' AND m.id_origen = 88
ORDER BY l.fecha_vencimiento;

-- 🔴 CP-09: el stock NUNCA debe quedar negativo
SELECT id_lote, cantidad_disponible FROM inventario_producto_terminado
WHERE cantidad_disponible < 0;
-- DEBE devolver 0 filas. Si devuelve alguna, el CHECK de V2 no está activo
-- y además el bloqueo FOR UPDATE no funcionó.

-- 🔴 CP-14: el precio histórico se congela
SELECT dp.id_detalle_pedido, dp.cantidad, dp.precio_unitario AS precio_pedido,
       p.precio AS precio_catalogo
FROM detalle_pedido dp JOIN productos p USING (id_producto)
WHERE dp.id_pedido = 47;
-- precio_pedido NO debe seguir a precio_catalogo

-- 🔴 CP-16/CP-17: el ajuste por diferencia deja UN movimiento, no dos
SELECT tipo_movimiento, cantidad, fecha FROM movimientos_inventario_pt
WHERE tabla_origen = 'detalle_pedido' AND id_origen = 88 ORDER BY id_movimiento;
-- Al subir de 10 a 15: un SALIDA_VENTA de 5.
-- Si aparece ENTRADA de 10 + SALIDA de 15, se implementó devolver-y-redescontar (R-07).

-- CP-08: la vista excluye vencidos
SELECT * FROM v_stock_producto_terminado WHERE id_producto = 3;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
