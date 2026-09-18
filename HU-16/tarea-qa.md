# HU-16 · Creación de pedidos — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01, CA-05 | Crear pedido válido | `{"idCliente":7,"fechaEntrega":"2026-09-20"}` | `201` con `idPedido`, `estado: "PENDIENTE"` | Positivo |
| CP-02 | CA-01 | Cliente inexistente | `{"idCliente":9999}` | `409 CLIENTE_INEXISTENTE` | Negativo |
| CP-03 | CA-01 | Cliente desactivado | Cliente con `estado = 0` | `409 CLIENTE_INACTIVO` | Negativo |
| CP-04 | CA-02 | 🔴 **La fecha del pedido la pone el servidor** | Crear sin enviar `fechaPedido` | `fechaPedido` = hoy, según el servidor | Positivo |
| CP-05 | CA-02 | 🔴 **Enviar `fechaPedido` no tiene efecto** | `{"fechaPedido":"2020-01-01"}` | Se ignora; queda la fecha de hoy (R-01) | Seguridad |
| CP-06 | CA-02 | 🔴 **Zona horaria** | Crear un pedido después de las 19:00 hora de Colombia | `fechaPedido` es **hoy**, no mañana (R-01) | Integridad |
| CP-07 | CA-03 | Sin fecha de entrega | Omitir `fechaEntrega` | `201`, `fechaEntrega: null` | Positivo |
| CP-08 | CA-03 | Entrega anterior al pedido | `fechaEntrega` de ayer | `400 FECHA_ENTREGA_INVALIDA` | Negativo |
| CP-09 | CA-03 | Entrega el mismo día | `fechaEntrega` = hoy | `201` — permitido (R-06) | Borde |
| CP-10 | CA-04 | Estado inicial | Crear pedido | Siempre `PENDIENTE`, aunque se envíe otro | Positivo |
| CP-11 | CA-04 | Transición válida | `PENDIENTE → EN_PREPARACION` | `200` | Positivo |
| CP-12 | CA-04 | `EN_PREPARACION → ENTREGADO` | `PATCH` | `200` | Positivo |
| CP-13 | CA-04 | 🔴 **`ENTREGADO` es terminal** | `ENTREGADO → PENDIENTE` | `409 TRANSICION_INVALIDA` (R-02) | Negativo |
| CP-14 | CA-04 | No se cancela un pedido entregado | `ENTREGADO → CANCELADO` | `409 TRANSICION_INVALIDA` | Negativo |
| CP-15 | CA-04 | Salto de estado | `PENDIENTE → ENTREGADO` | `409 TRANSICION_INVALIDA` | Negativo |
| CP-16 | — | 🔴 **Cancelar devuelve el stock** | Pedido con 2 líneas → `CANCELADO` | El stock vuelve a los lotes de origen (R-04) | Integridad |
| CP-17 | — | 🔴 **Se registran los movimientos de la devolución** | Tras CP-16 | Movimientos `ENTRADA_DEVOLUCION` en `movimientos_inventario_pt` | Integridad |
| CP-18 | — | Cancelar pedido sin líneas | Pedido vacío → `CANCELADO` | `200`; ningún movimiento generado | Borde |
| CP-19 | — | Editar pedido PENDIENTE | `PUT` sobre uno pendiente | `200` | Positivo |
| CP-20 | — | Editar pedido EN_PREPARACION | `PUT` sobre uno en preparación | `409 PEDIDO_NO_EDITABLE` (R-05) | Negativo |
| CP-21 | — | Cliente desactivado después | Crear, desactivar cliente, cambiar estado | `200` — el pedido sigue su curso (R-07) | Borde |
| CP-22 | — | Rol COMPRAS no puede crear pedidos | Token de COMPRAS | `403 SIN_PERMISO` | Seguridad |
| CP-23 | — | Rol PRODUCCION puede consultar | `GET` con token PRODUCCION | `200` | Positivo |

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

> **CP-06 es el caso que se descubre tarde.** Solo falla después de las 7 p.m. hora de
> Colombia, cuando el servidor está en UTC. Las pruebas de la mañana pasan sin problema y
> el defecto llega hasta la sustentación, donde aparece como "los pedidos de ayer salen con
> la fecha de hoy". Ejecutarlo requiere probar de noche o cambiar la hora del sistema.
>
> **CP-16 y CP-17 verifican R-04.** Es fácil implementar la cancelación como un simple
> `UPDATE estado` y olvidar la devolución de stock: el resultado es mercancía que el sistema
> da por vendida y que sigue en el estante.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-04 a CP-06: la fecha y la zona horaria
SELECT @@time_zone, @@system_time_zone, NOW(), CURDATE();
-- Se espera: America/Bogota · -05 · hora local de Colombia

SHOW CREATE TABLE pedidos\G
-- Se espera: `fecha_pedido` date NOT NULL DEFAULT (curdate())

SELECT id_pedido, fecha_pedido, fecha_entrega, estado
FROM pedidos ORDER BY id_pedido DESC LIMIT 1;

-- 🔴 CP-16 y CP-17: cancelar debe devolver el stock a los lotes de origen
SELECT ipt.id_lote, l.codigo_lote, ipt.cantidad_disponible
FROM inventario_producto_terminado ipt JOIN lotes l USING (id_lote)
WHERE l.id_producto = 3;
-- (anotar antes de cancelar, comparar después: debe haber subido)

SELECT tipo_movimiento, cantidad, saldo_anterior, saldo_posterior, tabla_origen, id_origen
FROM movimientos_inventario_pt
WHERE tabla_origen = 'pedidos' AND id_origen = 45 ORDER BY id_movimiento DESC;
-- ENTRADA_DEVOLUCION por cada línea del pedido cancelado

-- CP-13 a CP-15: el CHECK de estados de V2 §C-12
SHOW CREATE TABLE pedidos\G
-- CHECK (`estado` in ('PENDIENTE','EN_PREPARACION','ENTREGADO','CANCELADO'))
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
