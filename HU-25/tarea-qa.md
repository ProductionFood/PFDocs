# HU-25 · Consulta de pedidos — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Filtrar por cliente | `?idCliente=7` | Solo pedidos de ese cliente | Positivo |
| CP-02 | CA-01 | Cliente sin pedidos | Cliente nuevo | `content: []` | Borde |
| CP-03 | CA-02 | Filtrar por rango | `?fechaInicio=2026-09-01&fechaFin=2026-09-18` | Solo del rango | Positivo |
| CP-04 | CA-02 | 🔴 **Rango inclusivo** | Pedidos el 01/09 y el 18/09 exactos | **Ambos** aparecen (R-05) | Borde |
| CP-05 | CA-02 | Rango invertido | `inicio > fin` | `400 RANGO_FECHAS_INVALIDO` | Negativo |
| CP-06 | CA-03 | Filtrar por estado | `?estado=ENTREGADO` | Solo entregados | Positivo |
| CP-07 | CA-03 | Estado inválido | `?estado=DESPACHADO` | `400 PARAMETRO_INVALIDO` | Negativo |
| CP-08 | CA-01..03 | Filtros combinados | cliente + rango + estado | Los tres con `AND` | Positivo |
| CP-09 | CA-04 | Detalle del pedido | `?incluirDetalle=true` | Cada pedido con sus productos | Positivo |
| CP-10 | CA-04 | Tope de `size` con detalle | `?incluirDetalle=true&size=100` | Limitado a 20 o `400` (R-06) | Borde |
| CP-11 | CA-05 | Paginación | `?page=1&size=5` | Página 1 con máx. 5 | Positivo |
| CP-12 | CA-05 | Orden por total | `?sort=total,desc` | Ordenado descendente | Positivo |
| CP-13 | CA-05 | Orden fuera de lista blanca | `?sort=id_cliente,asc` | `400 CAMPO_ORDEN_INVALIDO` | Seguridad |
| CP-14 | — | 🔴 **El total usa el precio histórico** | Pedido a $1.200; subir catálogo a $1.500; consultar | El total **no cambia** (R-02) | Integridad |
| CP-15 | — | 🔴 **Las devoluciones se descuentan** | Pedido de 30; devolución aprobada de 3 | `totalNeto` = bruto − 3 unidades (R-04) | Integridad |
| CP-16 | — | Las devoluciones pendientes no descuentan | Devolución en estado `PENDIENTE` | `totalNeto` = `totalBruto` | Borde |
| CP-17 | — | 🔴 **Los cancelados no cuentan como venta** | Período con pedidos cancelados | `ventaNeta` los excluye (R-03) | Integridad |
| CP-18 | — | Comprometido separado de entregado | `GET /pedidos/resumen` | `entregados` y `enCurso` en cifras distintas | Positivo |
| CP-19 | — | Ticket promedio | 24 entregados por 1.840.000 | `76666.67` | Positivo |
| CP-20 | — | 🔴 **Ranking por unidades, no por importe** | Producto caro con 3 ventas vs. barato con 200 | El barato aparece primero (R-07) | Integridad |
| CP-21 | — | Ventas por cliente | `GET /pedidos/por-cliente` | Agrupado y ordenado por total | Positivo |
| CP-22 | — | Período sin pedidos | Rango vacío | Cifras en 0, sin error | Borde |
| CP-23 | — | ⏱ Rendimiento | 500 pedidos, `incluirDetalle=true&size=20` | < 2 s (RNF-02) | Rendimiento |
| CP-24 | — | Rol CONSULTA puede consultar | `GET /pedidos` con token CONSULTA | `200` | Positivo |
| CP-25 | — | Rol PRODUCCION no accede al resumen | `GET /pedidos/resumen` con token PRODUCCION | `403 SIN_PERMISO` | Seguridad |

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

> **CP-14 es la verificación de que la corrección §C-01 se está usando.** Es posible haber
> añadido la columna `precio_unitario` y seguir leyendo `productos.precio` en las consultas
> de reporte: la corrección estaría en el esquema pero no en el código, y el síntoma —un
> informe de ventas que cambia solo— aparecería meses después.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-14: el precio histórico. La prueba clave de §C-01.
SELECT p.id_pedido, dp.cantidad, dp.precio_unitario AS precio_venta,
       pr.precio AS precio_catalogo_hoy,
       dp.cantidad * dp.precio_unitario AS subtotal_historico
FROM pedidos p
  JOIN detalle_pedido dp USING (id_pedido)
  JOIN productos pr USING (id_producto)
WHERE p.id_pedido = 45;
-- precio_venta NO debe seguir a precio_catalogo_hoy.
-- Si el total de la API cambia al editar el catálogo, se está leyendo
-- productos.precio en lugar de detalle_pedido.precio_unitario.

-- 🔴 CP-15, CP-16: las devoluciones aprobadas descuentan
SELECT p.id_pedido,
       SUM(dp.cantidad * dp.precio_unitario) AS bruto,
       COALESCE(SUM(CASE WHEN d.estado = 'APROBADA'
                    THEN d.cantidad_devuelta * dp.precio_unitario END), 0) AS devuelto
FROM pedidos p
  JOIN detalle_pedido dp USING (id_pedido)
  LEFT JOIN devoluciones d USING (id_detalle_pedido)
WHERE p.id_pedido = 47
GROUP BY p.id_pedido;
-- neto = bruto - devuelto. Solo las APROBADA cuentan.

-- 🔴 CP-17: los cancelados fuera de la venta
SELECT p.estado, COUNT(DISTINCT p.id_pedido) AS pedidos,
       COALESCE(SUM(dp.cantidad * dp.precio_unitario), 0) AS total
FROM pedidos p LEFT JOIN detalle_pedido dp USING (id_pedido)
WHERE p.fecha_pedido BETWEEN '2026-09-01' AND '2026-09-30'
GROUP BY p.estado;

-- 🔴 CP-20: ranking por unidades
SELECT pr.nombre,
       SUM(dp.cantidad) AS unidades,
       SUM(dp.cantidad * dp.precio_unitario) AS importe,
       COUNT(DISTINCT p.id_pedido) AS pedidos
FROM detalle_pedido dp
  JOIN productos pr USING (id_producto)
  JOIN pedidos p USING (id_pedido)
WHERE p.estado = 'ENTREGADO'
  AND p.fecha_pedido BETWEEN '2026-09-01' AND '2026-09-30'
GROUP BY pr.id_producto, pr.nombre
ORDER BY unidades DESC;    -- por unidades, no por importe

-- CP-23: el índice de V2
EXPLAIN SELECT id_pedido FROM pedidos
WHERE fecha_pedido BETWEEN '2026-09-01' AND '2026-09-30' AND estado = 'ENTREGADO';
-- Se espera: key = idx_pedidos_fecha
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
