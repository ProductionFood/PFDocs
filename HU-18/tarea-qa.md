# HU-18 · Registro de devoluciones — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..06 | Registrar devolución válida | 3 de una línea de 30 | `201`, `estado: "PENDIENTE"` | Positivo |
| CP-02 | CA-01 | Detalle de pedido inexistente | `{"idDetallePedido":9999}` | `409 DETALLE_PEDIDO_INEXISTENTE` | Negativo |
| CP-03 | CA-02 | Motivo vacío | `{"motivo":""}` | `400 VALIDACION_FALLIDA` | Negativo |
| CP-04 | CA-02 | Motivo de 51 caracteres | `"A"×51` | `400` | Borde |
| CP-05 | CA-03 | Sin descripción | Omitir `descripcion` | `201`, `descripcion: null` | Positivo |
| CP-06 | CA-03 | Descripción de 256 caracteres | `"A"×256` | `400` | Borde |
| CP-07 | CA-04 | 🔴 **No se puede devolver más de lo pedido** | Devolver 31 de una línea de 30 | `422 CANTIDAD_EXCEDE_PEDIDO` | Negativo |
| CP-08 | CA-04 | Devolver exactamente lo pedido | 30 de 30 | `201` | Borde |
| CP-09 | CA-04 | 🔴 **El acumulado incluye las pendientes** | Línea de 10; devolución pendiente de 8; registrar otra de 5 | `422` — solo quedan 2 disponibles (R-03) | Integridad |
| CP-10 | CA-04 | Las rechazadas liberan cantidad | Línea de 10; devolución de 8 **rechazada**; registrar 8 | `201` — las rechazadas no cuentan (R-03) | Borde |
| CP-11 | CA-04 | Cantidad cero | `{"cantidadDevuelta":0}` | `400 CANTIDAD_INVALIDA` | Negativo |
| CP-12 | CA-05 | 🔴 **La fecha la pone el servidor** | Crear sin `fechaDevolucion` | Fecha de hoy según el servidor (R-06) | Positivo |
| CP-13 | CA-05 | Enviar fecha no tiene efecto | `{"fechaDevolucion":"2020-01-01"}` | Se ignora; queda hoy | Seguridad |
| CP-14 | CA-06 | Estado inicial | Crear devolución | Siempre `PENDIENTE` | Positivo |
| CP-15 | CA-07 | 🔴 **Aprobar repone el stock** | `PATCH` a `APROBADA` con cantidad 3 | El inventario del lote sube 3 (R-01) | Integridad |
| CP-16 | CA-07 | 🔴 **Registrar NO repone stock** | Crear devolución y revisar inventario | El stock **no cambia** hasta aprobar (R-01) | Integridad |
| CP-17 | CA-07 | 🔴 **Vuelve al lote correcto** | Línea que consumió 2 lotes; aprobar devolución | Repone en los lotes del kardex, en orden inverso (R-02) | Integridad |
| CP-18 | CA-07 | 🔴 **Se registra el movimiento** | Tras CP-15 | `ENTRADA_DEVOLUCION` con origen `devoluciones` | Integridad |
| CP-19 | — | Rechazar no repone stock | `PATCH` a `RECHAZADA` | `200`; el inventario no cambia | Positivo |
| CP-20 | — | 🔴 **Aprobar dos veces** | `PATCH` a `APROBADA` sobre una ya aprobada | `409 TRANSICION_INVALIDA`; el stock no sube otra vez | Integridad |
| CP-21 | — | Aprobada no vuelve a pendiente | `APROBADA → PENDIENTE` | `409 TRANSICION_INVALIDA` (R-04) | Negativo |
| CP-22 | — | Devolución sobre pedido PENDIENTE | Pedido sin entregar | `409 PEDIDO_NO_DEVOLVIBLE` (R-05) | Negativo |
| CP-23 | — | Devolución sobre pedido CANCELADO | Pedido cancelado | `409 PEDIDO_NO_DEVOLVIBLE` — el stock ya volvió | Negativo |
| CP-24 | — | Transaccionalidad de la aprobación | Provocar fallo al reponer | La devolución **no** queda aprobada | Integridad |
| CP-25 | — | Rol COMPRAS no puede aprobar | Token de COMPRAS | `403 SIN_PERMISO` | Seguridad |

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

> **CP-09, CP-16 y CP-20 son los casos críticos.**
>
> **CP-09:** si el acumulado solo cuenta las aprobadas, se pueden registrar varias
> devoluciones pendientes que sumadas superan lo vendido; al aprobarlas todas, el inventario
> recibe más unidades de las que salieron.
>
> **CP-16:** reponer al registrar en lugar de al aprobar devuelve al estante producto que
> todavía no se ha revisado — y el motivo más común de devolución es que está en mal estado.
>
> **CP-20:** sin validar la transición antes de reponer, dos clics en "Aprobar" duplican la
> reposición.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-15 a CP-18: el stock vuelve al aprobar, y al lote correcto
-- Antes de aprobar:
SELECT l.codigo_lote, l.fecha_vencimiento, ipt.cantidad_disponible
FROM lotes l JOIN inventario_producto_terminado ipt USING (id_lote)
WHERE l.id_producto = 3 ORDER BY l.fecha_vencimiento;

-- De qué lotes salió esa línea de pedido (esto es lo que debe consultar el backend)
SELECT m.id_lote, l.codigo_lote, m.cantidad, m.id_movimiento
FROM movimientos_inventario_pt m JOIN lotes l USING (id_lote)
WHERE m.tabla_origen = 'detalle_pedido' AND m.id_origen = 88
  AND m.tipo_movimiento = 'SALIDA_VENTA'
ORDER BY m.id_movimiento DESC;   -- orden inverso al consumo (R-02)

-- (ejecutar PATCH /devoluciones/15/estado {"estado":"APROBADA"})

-- Después: el saldo subió en el lote correcto
SELECT tipo_movimiento, cantidad, saldo_anterior, saldo_posterior, tabla_origen, id_origen
FROM movimientos_inventario_pt
WHERE tabla_origen = 'devoluciones' ORDER BY id_movimiento DESC LIMIT 3;

-- 🔴 CP-09: el acumulado debe incluir las PENDIENTE
SELECT dp.id_detalle_pedido, dp.cantidad AS pedida,
       COALESCE(SUM(CASE WHEN d.estado IN ('APROBADA','PENDIENTE')
                         THEN d.cantidad_devuelta END), 0) AS comprometida,
       COALESCE(SUM(CASE WHEN d.estado = 'RECHAZADA'
                         THEN d.cantidad_devuelta END), 0) AS rechazada
FROM detalle_pedido dp LEFT JOIN devoluciones d USING (id_detalle_pedido)
WHERE dp.id_detalle_pedido = 88
GROUP BY dp.id_detalle_pedido, dp.cantidad;
-- "comprometida" NUNCA debe superar "pedida"

-- 🔴 CP-20: aprobar dos veces no debe generar dos reposiciones
SELECT COUNT(*) FROM movimientos_inventario_pt
WHERE tabla_origen = 'devoluciones' AND id_origen = 15;
-- Debe ser igual al número de lotes repuestos, nunca el doble

-- CP-12: el DEFAULT de V2 §C-07
SHOW CREATE TABLE devoluciones\G
-- `fecha_devolucion` date NOT NULL DEFAULT (curdate())
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
