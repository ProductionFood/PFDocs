# HU-10 · Consulta de inventario — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Listar el inventario | `GET /inventario` | `200` con todas las materias primas y su disponible | Positivo |
| CP-02 | CA-01 | Materia prima sin movimientos aparece en cero | MP recién creada | Aparece con `cantidadDisponible: 0.00` (R-02) | Borde |
| CP-03 | CA-02 | Se muestra la fecha de actualización | Revisar CP-01 | `fechaActualizacion` presente y con formato ISO | Positivo |
| CP-04 | CA-02 | La fecha cambia al mover el stock | Registrar un lote (HU-09) y volver a consultar | `fechaActualizacion` es posterior | Integridad |
| CP-05 | CA-03 | Filtrar por materia prima | `?idMateriaPrima=5` | Solo esa materia prima | Positivo |
| CP-06 | CA-04 | Filtrar solo stock bajo | `?soloStockBajo=true` | Solo `BAJO` y `AGOTADO` | Positivo |
| CP-07 | CA-04 | Stock exactamente igual al mínimo | disponible = 50, mínimo = 50 | `estadoStock: "OK"`, **no** aparece en stock bajo | Borde |
| CP-08 | CA-04 | Stock una unidad por debajo | disponible = 49,99, mínimo = 50 | `estadoStock: "BAJO"`, `faltante: 0.01` | Borde |
| CP-09 | CA-04 | Stock en cero | disponible = 0 | `estadoStock: "AGOTADO"` | Borde |
| CP-10 | CA-04 | Stock mínimo en cero | mínimo = 0, disponible = 0 | `estadoStock: "AGOTADO"`; con disponible > 0 → `OK` (R-05) | Borde |
| CP-11 | — | Cálculo del faltante | disponible = 12,50, mínimo = 50 | `faltante: 37.50` | Positivo |
| CP-12 | — | Materia prima inactiva con stock | Desactivar una con stock > 0 | Sigue apareciendo, marcada inactiva (R-04) | Borde |
| CP-13 | — | Kardex de una materia prima | `GET /inventario/5/movimientos` | Movimientos ordenados de más reciente a más antiguo | Positivo |
| CP-14 | — | 🔴 **Coherencia saldo ↔ kardex** | Comparar saldo y último movimiento | `saldo_posterior` = `cantidad_disponible` (R-03) | Integridad |
| CP-15 | — | Kardex de MP sin movimientos | MP recién creada | `content: []`, sin error | Borde |
| CP-16 | — | Movimiento del sistema | Movimiento con `id_usuario = NULL` | Se muestra "Sistema", no error ni usuario 0 | Borde |
| CP-17 | — | **No existe endpoint de escritura** | `POST`/`PUT`/`PATCH /inventario` | `405` o `404` (R-01) | Seguridad |
| CP-18 | — | Rol CONSULTA puede ver | `GET` con token CONSULTA | `200` | Positivo |
| CP-19 | — | Rol VENTAS no accede al inventario de MP | `GET` con token VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-20 | — | Orden fuera de lista blanca | `?sort=costo_unitario,asc` | `400 CAMPO_ORDEN_INVALIDO` | Seguridad |

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

> **CP-14 debe ejecutarse al cerrar cada fase**, no solo en esta historia. Es la prueba que
> detecta si algún módulo posterior actualiza el inventario sin pasar por
> `InventarioService`. Mientras devuelva 0 filas, el kardex es confiable; en cuanto devuelva
> una, el saldo dejó de ser explicable.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-14: la conciliación. DEBE devolver 0 filas, siempre.
SELECT i.id_materia_prima, i.cantidad_disponible, m.saldo_posterior
FROM inventario i
LEFT JOIN (
    SELECT id_materia_prima, saldo_posterior,
           ROW_NUMBER() OVER (PARTITION BY id_materia_prima ORDER BY id_movimiento DESC) rn
    FROM movimientos_inventario_mp
) m ON m.id_materia_prima = i.id_materia_prima AND m.rn = 1
WHERE i.cantidad_disponible <> COALESCE(m.saldo_posterior, 0);
-- Una fila aquí significa que alguna ruta de código actualizó el saldo sin registrar
-- el movimiento: exactamente lo que el kardex existe para impedir.

-- CP-02: toda materia prima debe tener fila de inventario (HU-08 R-01)
SELECT mp.nombre FROM materias_primas mp
LEFT JOIN inventario i USING (id_materia_prima)
WHERE i.id_inventario IS NULL;   -- 0 filas

-- CP-07/CP-08: verificar el cálculo de la vista
SELECT nombre, cantidad_disponible, stock_minimo, estado_stock, faltante
FROM v_inventario_materia_prima ORDER BY estado_stock, faltante DESC;

-- CP-04: el ON UPDATE de V2 §C-07 mantiene la fecha sola
SHOW CREATE TABLE inventario\G
-- `fecha_actualizacion` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP
--                                ON UPDATE CURRENT_TIMESTAMP
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
