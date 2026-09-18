# HU-09 · Lotes de materia prima — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..04 | Registrar lote válido | `{"codigoLote":"H-2026-0912","idMateriaPrima":5,"fechaProduccion":"2026-09-10","fechaVencimiento":"2027-03-10","cantidad":100}` | `201` con `idLote` y saldos | Positivo |
| CP-02 | — | 🔴 **El inventario sube exactamente la cantidad** | Saldo antes 12,50 → registrar 100 | Saldo después = 112,50 (R-01) | Integridad |
| CP-03 | — | 🔴 **Se registra el movimiento de kardex** | Tras CP-01 | Fila `ENTRADA_LOTE` con saldo anterior 12,50 y posterior 112,50 | Integridad |
| CP-04 | — | 🔴 **Lote con `idDetalleCompra` NO suma** | Registrar lote asociado a una compra recibida | El inventario **no cambia** (R-02) | Integridad |
| CP-05 | CA-01 | Código duplicado | Repetir `codigoLote` | `409 CODIGO_LOTE_DUPLICADO` | Negativo |
| CP-06 | CA-01 | Código duplicado con otra capitalización | `"h-2026-0912"` existiendo `"H-2026-0912"` | `409` — se normaliza a mayúsculas (R-06) | Borde |
| CP-07 | CA-01 | Código de 31 caracteres | `"A"×31` | `400` | Borde |
| CP-08 | CA-02 | Materia prima inexistente | `{"idMateriaPrima":9999}` | `409 MATERIA_PRIMA_INEXISTENTE` | Negativo |
| CP-09 | CA-03 | Vencimiento anterior a producción | `produccion:"2026-09-10"`, `vencimiento:"2026-09-01"` | `400 FECHAS_INVALIDAS` | Negativo |
| CP-10 | CA-03 | Vencimiento igual a producción | Ambas `"2026-09-10"` | `201` — permitido (R-04) | Borde |
| CP-11 | CA-03 | Fecha omitida | Sin `fechaVencimiento` | `400` | Negativo |
| CP-12 | CA-04 | Cantidad cero | `{"cantidad":0}` | `400 CANTIDAD_INVALIDA` — `CHECK > 0` de V2 | Negativo |
| CP-13 | CA-04 | Cantidad negativa | `{"cantidad":-10}` | `400` | Negativo |
| CP-14 | CA-05 | Listar por materia prima | `?idMateriaPrima=5` | Solo lotes de esa materia prima | Positivo |
| CP-15 | — | Lote ya vencido | `fechaVencimiento` en el pasado | `201` con `vencido: true` (R-05); **sí suma** al inventario | Borde |
| CP-16 | — | Filtro de próximos a vencer | `?proximosAVencer=7` | Solo los que vencen en ≤ 7 días | Positivo |
| CP-17 | — | **El lote no puede tener producto y materia prima a la vez** | Intentar insertar ambos por SQL directo | La base lo rechaza — `ck_lotes_exclusividad` (R-03) | Integridad |
| CP-18 | — | Transacción: si falla el kardex, no sube el inventario | Provocar fallo en el `INSERT` del movimiento | El saldo de `inventario` **no cambia** | Integridad |
| CP-19 | — | Rol VENTAS no puede registrar | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-20 | — | Rol PRODUCCION puede registrar | Token de PRODUCCION | `201` | Positivo |

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

> **CP-04 es el caso más importante de esta historia.** Verifica la protección contra la
> doble suma (R-02): si un lote asociado a una compra ya recibida vuelve a sumar, el
> inventario marca el doble de existencias y **nada en el sistema lo señala**. El error se
> descubre cuando alguien va a la bodega a buscar 200 kg de harina y encuentra 100.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-02 y CP-03: el saldo y el kardex deben coincidir SIEMPRE
SELECT cantidad_disponible FROM inventario WHERE id_materia_prima = 5;

SELECT tipo_movimiento, cantidad, saldo_anterior, saldo_posterior, tabla_origen, id_origen
FROM movimientos_inventario_mp
WHERE id_materia_prima = 5 ORDER BY id_movimiento DESC LIMIT 1;
-- ENTRADA_LOTE | 100.000 | 12.50 | 112.50 | lotes | 14
-- El saldo_posterior DEBE ser igual a inventario.cantidad_disponible

-- Conciliación general (05-ESTANDARES-QA.md §5). DEBE devolver 0 filas.
SELECT i.id_materia_prima, i.cantidad_disponible, m.saldo_posterior
FROM inventario i
LEFT JOIN (
    SELECT id_materia_prima, saldo_posterior,
           ROW_NUMBER() OVER (PARTITION BY id_materia_prima ORDER BY id_movimiento DESC) rn
    FROM movimientos_inventario_mp
) m ON m.id_materia_prima = i.id_materia_prima AND m.rn = 1
WHERE i.cantidad_disponible <> COALESCE(m.saldo_posterior, 0);

-- CP-17: el CHECK de exclusividad de V2 §C-10
INSERT INTO lotes (codigo_lote, id_producto, id_materia_prima, fecha_produccion, fecha_vencimiento, cantidad)
VALUES ('X-1', 1, 5, curdate(), curdate(), 10);
-- DEBE fallar con error 3819
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
