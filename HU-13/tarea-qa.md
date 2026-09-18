# HU-13 · Registro de compras — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..04 | Crear compra válida | `{"idProveedor":3,"fecha":"2026-09-18","estado":"PENDIENTE"}` | `201` con `idCompra` | Positivo |
| CP-02 | CA-01 | Proveedor inexistente | `{"idProveedor":9999}` | `409 PROVEEDOR_INEXISTENTE` | Negativo |
| CP-03 | CA-01 | Proveedor desactivado | Proveedor con `estado = 0` | `409 PROVEEDOR_INACTIVO` | Negativo |
| CP-04 | CA-02 | Fecha omitida | Sin `fecha` | `400` o se asigna hoy por defecto | Borde |
| CP-05 | CA-02 | Fecha futura | `"2027-01-01"` | `400 FECHA_FUTURA` (R-07) | Negativo |
| CP-06 | CA-02 | Fecha pasada | `"2026-08-01"` | `201` — permitido (R-07) | Borde |
| CP-07 | CA-03 | Estado inválido | `{"estado":"ENVIADA"}` | `400` — fuera de los valores canónicos | Negativo |
| CP-08 | CA-03 | Estado en minúsculas | `{"estado":"pendiente"}` | `400` o se normaliza; nunca se guarda en minúsculas | Borde |
| CP-09 | — | 🔴 **Recibir una compra suma al inventario** | Compra con 2 líneas; `PATCH` a `RECIBIDA` | `200`; ambos inventarios suben exactamente (R-01) | Integridad |
| CP-10 | — | 🔴 **Se generan los movimientos de kardex** | Tras CP-09 | 2 movimientos `ENTRADA_COMPRA` con origen `detalle_compra` | Integridad |
| CP-11 | — | 🔴 **`RECIBIDA` no vuelve a `PENDIENTE`** | `PATCH` de `RECIBIDA` a `PENDIENTE` | `409 TRANSICION_INVALIDA` (R-02) | Negativo |
| CP-12 | — | No se puede cancelar una compra recibida | `PATCH` de `RECIBIDA` a `CANCELADA` | `409 TRANSICION_INVALIDA` (R-05) | Negativo |
| CP-13 | — | Cancelar desde PENDIENTE | `PATCH` a `CANCELADA` | `200`; **el inventario no cambia** | Positivo |
| CP-14 | — | Recibir compra sin detalle | Compra sin líneas → `RECIBIDA` | `409 COMPRA_SIN_DETALLE` (R-03) | Negativo |
| CP-15 | — | 🔴 **Recibir dos veces no suma dos veces** | `PATCH` a `RECIBIDA` sobre una ya recibida | `409 TRANSICION_INVALIDA`; el inventario no cambia | Integridad |
| CP-16 | — | 🔴 **Transaccionalidad de la recepción** | Compra de 3 líneas; provocar fallo en la tercera | **Ninguna** de las tres entra al inventario (R-01) | Integridad |
| CP-17 | — | Editar compra PENDIENTE | `PUT` sobre una pendiente | `200` | Positivo |
| CP-18 | — | Editar compra RECIBIDA | `PUT` sobre una recibida | `409 COMPRA_NO_EDITABLE` (R-04) | Negativo |
| CP-19 | — | Compra de proveedor desactivado después | Crear, desactivar proveedor, recibir | `200` — la recepción procede (R-06) | Borde |
| CP-20 | — | Rol VENTAS no puede crear compras | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-21 | — | Rol PRODUCCION no puede recibir | `PATCH` con token PRODUCCION | `403 SIN_PERMISO` | Seguridad |
| CP-22 | — | Rol COMPRAS puede recibir | `PATCH` con token COMPRAS | `200` | Positivo |

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

> **CP-15 y CP-16 son los casos críticos de esta historia.**
>
> CP-15 protege contra la doble suma por doble recepción: si el estado no se valida antes
> de sumar, dos clics en "Recibir" duplican el stock sin ningún mensaje.
>
> CP-16 verifica que la transacción envuelve **toda** la recepción. Una recepción parcial
> —tres líneas, dos aplicadas— deja el inventario en un estado que ni el operador ni el
> kardex pueden explicar, y que no se puede deshacer porque `RECIBIDA` es terminal.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-09 y CP-10: antes de recibir
SELECT id_materia_prima, cantidad_disponible FROM inventario WHERE id_materia_prima IN (5,6);
-- 5 → 12.50 ; 6 → 45.00

-- (ejecutar PATCH /compras/12/estado {"estado":"RECIBIDA"} con 100 kg y 50 kg)

-- después
SELECT id_materia_prima, cantidad_disponible FROM inventario WHERE id_materia_prima IN (5,6);
-- 5 → 112.50 ; 6 → 95.00

SELECT tipo_movimiento, cantidad, saldo_anterior, saldo_posterior, tabla_origen, id_origen
FROM movimientos_inventario_mp
WHERE tabla_origen = 'detalle_compra' ORDER BY id_movimiento DESC LIMIT 2;
-- ENTRADA_COMPRA con los saldos correctos y el id de cada línea

-- 🔴 CP-16: tras el fallo forzado, NINGUNA entrada debe haberse aplicado
SELECT id_materia_prima, cantidad_disponible FROM inventario WHERE id_materia_prima IN (5,6,7);
-- Idénticos a los valores previos

-- CP-15: una compra recibida no puede generar movimientos nuevos
SELECT COUNT(*) FROM movimientos_inventario_mp
WHERE tabla_origen = 'detalle_compra'
  AND id_origen IN (SELECT id_detalle_compra FROM detalle_compra WHERE id_compra = 12);
-- Debe ser igual al número de líneas, nunca el doble

-- Conciliación general (05-ESTANDARES-QA.md §5): 0 filas
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
