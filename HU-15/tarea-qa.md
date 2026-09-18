# HU-15 · Historial de compras — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Filtrar por proveedor | `?idProveedor=3` | Solo compras de ese proveedor | Positivo |
| CP-02 | CA-01 | Proveedor sin compras | Proveedor recién creado | `content: []`, `totalElements: 0` | Borde |
| CP-03 | CA-02 | Filtrar por rango | `?fechaInicio=2026-09-01&fechaFin=2026-09-18` | Solo compras del rango | Positivo |
| CP-04 | CA-02 | 🔴 **Rango inclusivo en ambos extremos** | Compras el 01/09 y el 18/09 exactos | **Ambas** aparecen (R-03) | Borde |
| CP-05 | CA-02 | Rango invertido | `fechaInicio=18/09`, `fechaFin=01/09` | `400 RANGO_FECHAS_INVALIDO` | Negativo |
| CP-06 | CA-02 | Rango de un solo día | `fechaInicio = fechaFin = 2026-09-18` | Compras de ese día | Borde |
| CP-07 | CA-03 | Filtrar por estado | `?estado=RECIBIDA` | Solo recibidas | Positivo |
| CP-08 | CA-03 | Estado inválido | `?estado=ENVIADA` | `400 PARAMETRO_INVALIDO` | Negativo |
| CP-09 | CA-01..03 | Filtros combinados | proveedor + rango + estado | Se aplican los tres con `AND` (R-02) | Positivo |
| CP-10 | CA-04 | Detalle en el listado | `?incluirDetalle=true` | Cada compra trae sus líneas | Positivo |
| CP-11 | CA-04 | Subtotales correctos | Revisar CP-10 | `subtotal = cantidad × precio` en cada línea | Positivo |
| CP-12 | CA-04 | Tope de `size` con detalle | `?incluirDetalle=true&size=100` | `size` limitado a 20 o `400` (R-06) | Borde |
| CP-13 | — | 🔴 **Las canceladas no cuentan como gasto** | Período con una compra cancelada | `gastoEjecutado` la excluye (R-04) | Integridad |
| CP-14 | — | Pendientes separadas de recibidas | `GET /compras/resumen` | `gastoEjecutado` y `gastoComprometido` en cifras distintas | Positivo |
| CP-15 | — | Gasto por proveedor | `GET /compras/por-proveedor` | Agrupado, ordenado por total descendente | Positivo |
| CP-16 | — | Porcentajes suman 100 | Revisar CP-15 | La suma de `porcentaje` es 100,00 ± 0,01 | Positivo |
| CP-17 | — | Resumen de período sin compras | Rango sin datos | Totales en `0.00`, sin error | Borde |
| CP-18 | — | ⏱ Rendimiento con detalle | 200 compras, `incluirDetalle=true&size=20` | Responde en < 2 s (RNF-02) — verifica que no hay N+1 | Rendimiento |
| CP-19 | — | Rol CONSULTA puede ver el historial | `GET /compras` con token CONSULTA | `200` | Positivo |
| CP-20 | — | Rol CONSULTA no accede al resumen | `GET /compras/resumen` con token CONSULTA | `403 SIN_PERMISO` | Seguridad |
| CP-21 | — | **No hay endpoints de escritura aquí** | `POST /compras/resumen` | `405` o `404` (R-01) | Seguridad |

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

> **CP-18 es la prueba que detecta el problema N+1** (R-06). Si el historial con detalle
> tarda más de dos segundos con 200 compras, el backend está haciendo una consulta por
> compra en lugar de un `JOIN` con agrupación. El síntoma aparece solo con datos reales:
> con las cinco compras de prueba, ambas implementaciones parecen igual de rápidas.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-13: las canceladas fuera del gasto ejecutado
SELECT c.estado, COUNT(*) AS compras,
       COALESCE(SUM(d.cantidad * d.precio_unitario), 0) AS total
FROM compras c LEFT JOIN detalle_compra d USING (id_compra)
WHERE c.fecha BETWEEN '2026-09-01' AND '2026-09-30'
GROUP BY c.estado;
-- gastoEjecutado de la API = total de RECIBIDA, sin CANCELADA

-- CP-04: el rango debe ser inclusivo
SELECT id_compra, fecha FROM compras
WHERE fecha BETWEEN '2026-09-01' AND '2026-09-18' ORDER BY fecha;
-- Las compras del 01 y del 18 deben estar presentes

-- CP-15/CP-16: gasto por proveedor
SELECT p.nombre, COUNT(DISTINCT c.id_compra) AS compras,
       SUM(d.cantidad * d.precio_unitario) AS total
FROM compras c
  JOIN proveedores p USING (id_proveedor)
  JOIN detalle_compra d USING (id_compra)
WHERE c.estado = 'RECIBIDA' AND c.fecha BETWEEN '2026-09-01' AND '2026-09-30'
GROUP BY p.id_proveedor, p.nombre ORDER BY total DESC;

-- CP-18: el índice de V2 debe usarse
EXPLAIN SELECT id_compra FROM compras
WHERE fecha BETWEEN '2026-09-01' AND '2026-09-30' AND estado = 'RECIBIDA';
-- Se espera: key = idx_compras_fecha
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
