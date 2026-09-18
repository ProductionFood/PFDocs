# HU-24 · Dashboard resumen — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Pedidos del día | 3 pedidos hoy, 5 ayer | `pedidosHoy.cantidad = 3` | Positivo |
| CP-02 | CA-01 | 🔴 **Zona horaria** | Crear un pedido a las 20:00 hora de Colombia | Cuenta como **hoy**, no mañana (R-01) | Integridad |
| CP-03 | CA-01 | Total de ventas del día | 3 pedidos por 284.500 | `pedidosHoy.total = 284500.00` (R-06) | Positivo |
| CP-04 | CA-01 | Sin pedidos hoy | Día sin registros | `cantidad: 0`, `total: 0.00`, sin error | Borde |
| CP-05 | CA-02 | Compras pendientes | 2 pendientes, 5 recibidas | `comprasPendientes.cantidad = 2` | Positivo |
| CP-06 | CA-02 | 🔴 **Sin filtro de fecha** | Compra pendiente de hace un mes | **Aparece** en el conteo (R-02) | Integridad |
| CP-07 | CA-02 | Las canceladas no cuentan | 1 pendiente, 1 cancelada | Solo cuenta la pendiente | Borde |
| CP-08 | CA-03 | Stock bajo | 2 bajo mínimo, 1 agotada | `cantidad: 3`, `agotadas: 1` | Positivo |
| CP-09 | CA-03 | Stock exactamente en el mínimo | disponible = mínimo | **No** aparece como bajo | Borde |
| CP-10 | CA-04 | Producción del día | 2 planes para hoy | `produccionHoy.cantidad = 2` | Positivo |
| CP-11 | CA-04 | Los planes cancelados | Plan de hoy cancelado | Se cuenta aparte en `porEstado` | Borde |
| CP-12 | CA-05 | Próximos a vencer | Lotes que vencen en 3 y 5 días | Ambos en `porVencer` | Positivo |
| CP-13 | CA-05 | Vence exactamente en 7 días | `fecha_vencimiento = hoy + 7` | **Incluido** — rango inclusivo | Borde |
| CP-14 | CA-05 | Vence en 8 días | `hoy + 8` | **No incluido** | Borde |
| CP-15 | CA-05 | 🔴 **Solo cuentan los que tienen stock** | Lote por vencer con 0 unidades | **No aparece** (R-03) | Integridad |
| CP-16 | CA-05 | Lote ya vencido con stock | Venció hace 2 días, quedan 8 unidades | **Aparece** con `diasParaVencer: -2` (R-03) | Borde |
| CP-17 | — | ⏱ **Rendimiento del dashboard completo** | Base con 500 pedidos, 200 compras, 100 lotes | `GET /dashboard` en < 2 s (RNF-02, R-04) | Rendimiento |
| CP-18 | — | Base vacía | Sistema recién instalado | Todas las cifras en 0, sin errores ni nulos | Borde |
| CP-19 | — | Rol VENTAS accede | `GET /dashboard` con token VENTAS | `200` | Positivo |
| CP-20 | — | Rol CONSULTA accede | `GET /dashboard` con token CONSULTA | `200` | Positivo |
| CP-21 | — | **Sin endpoints de escritura** | `POST /dashboard` | `405` o `404` | Seguridad |

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

> **CP-02 es el caso que hace visible el problema de zona horaria de todo el sistema.**
> Si el servidor está en UTC, a partir de las 7 p.m. hora de Colombia el dashboard muestra
> "0 pedidos hoy" mientras el equipo acaba de registrar varios. Es el síntoma más inmediato
> del defecto que también afecta a HU-16 y HU-18.
>
> **CP-17 conviene ejecutarlo con datos realistas.** Con las diez filas de prueba todo
> responde al instante; el problema aparece con volumen, y es justo la pantalla que todos
> abren al entrar.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-02: la zona horaria. Ejecutar antes de cualquier prueba del dashboard.
SELECT @@time_zone, @@system_time_zone, NOW(), CURDATE();
-- Se espera: America/Bogota · -05 · fecha y hora locales de Colombia

-- CP-01, CP-03: pedidos del día
SELECT COUNT(DISTINCT p.id_pedido) AS pedidos,
       COALESCE(SUM(dp.cantidad * dp.precio_unitario), 0) AS total
FROM pedidos p LEFT JOIN detalle_pedido dp USING (id_pedido)
WHERE p.fecha_pedido = curdate();

-- 🔴 CP-06: compras pendientes, SIN filtro de fecha
SELECT COUNT(*) AS pendientes, MIN(fecha) AS mas_antigua,
       DATEDIFF(curdate(), MIN(fecha)) AS dias
FROM compras WHERE estado = 'PENDIENTE';

-- CP-08, CP-09: stock bajo desde la vista de V2
SELECT nombre, cantidad_disponible, stock_minimo, estado_stock, faltante
FROM v_inventario_materia_prima
WHERE estado_stock IN ('BAJO','AGOTADO') ORDER BY faltante DESC;

-- CP-10: producción del día
SELECT estado, COUNT(*) FROM planes_produccion
WHERE fecha_produccion = curdate() GROUP BY estado;

-- 🔴 CP-12 a CP-16: por vencer, SOLO con stock
SELECT l.codigo_lote, p.nombre, l.fecha_vencimiento,
       DATEDIFF(l.fecha_vencimiento, curdate()) AS dias,
       ipt.cantidad_disponible
FROM lotes l
  JOIN productos p USING (id_producto)
  JOIN inventario_producto_terminado ipt USING (id_lote)
WHERE ipt.cantidad_disponible > 0
  AND l.fecha_vencimiento <= curdate() + INTERVAL 7 DAY
ORDER BY l.fecha_vencimiento;
-- Incluye los vencidos (dias negativos) y excluye los agotados

-- 🔴 CP-17: verificar que se usan los índices de V2 §C-15
EXPLAIN SELECT COUNT(*) FROM pedidos WHERE fecha_pedido = curdate();
EXPLAIN SELECT COUNT(*) FROM compras WHERE estado = 'PENDIENTE';
EXPLAIN SELECT COUNT(*) FROM planes_produccion WHERE fecha_produccion = curdate();
EXPLAIN SELECT * FROM lotes WHERE fecha_vencimiento <= curdate() + INTERVAL 7 DAY;
-- Se esperan: idx_pedidos_fecha, idx_compras_fecha, idx_planes_fecha, idx_lotes_vencimiento
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
