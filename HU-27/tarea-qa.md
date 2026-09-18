# HU-27 · Alerta de stock bajo — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Listar alertas | 2 bajo mínimo, 1 agotada | `200` con 3 ítems | Positivo |
| CP-02 | CA-01 | 🔴 **Stock exactamente igual al mínimo** | disponible 50, mínimo 50 | **No aparece** — la comparación es `<` (R-02) | Borde |
| CP-03 | CA-01 | Una unidad por debajo | disponible 49,99, mínimo 50 | **Aparece** con `faltante: 0.01` | Borde |
| CP-04 | CA-01 | Stock por encima | disponible 96, mínimo 50 | No aparece | Positivo |
| CP-05 | CA-02 | Se muestran los tres datos | Revisar CP-01 | Nombre, disponible y mínimo presentes | Positivo |
| CP-06 | CA-03 | 🔴 **Cálculo del faltante** | disponible 46,30, mínimo 50 | `faltante: 3.70` | Positivo |
| CP-07 | CA-03 | Faltante con stock en cero | disponible 0, mínimo 5 | `faltante: 5.00` | Borde |
| CP-08 | CA-04 | 🔴 **Orden por urgencia** | Agotada (falta 5) + baja (falta 40) | La **agotada primero**, aunque falte menos (R-03) | Integridad |
| CP-09 | CA-04 | Orden dentro del mismo nivel | Dos `BAJO`: faltan 15,5 y 3,7 | La de 15,5 primero (CA-04) | Positivo |
| CP-10 | — | 🔴 **`stock_minimo = 0` no genera alerta BAJO** | mínimo 0, disponible 10 | No aparece (R-04) | Borde |
| CP-11 | — | Mínimo cero y disponible cero | mínimo 0, disponible 0 | **Aparece** como `AGOTADO` (R-04) | Borde |
| CP-12 | — | 🔴 **Las inactivas se excluyen** | MP desactivada bajo mínimo | No aparece por defecto (R-05) | Integridad |
| CP-13 | — | Incluir inactivas a petición | `?incluirInactivas=true` | Aparece | Positivo |
| CP-14 | — | Materia prima sin fila de inventario | MP creada sin inventario (defecto HU-08) | Aparece como `AGOTADO` con disponible 0 | Borde |
| CP-15 | — | Costo de reposición | faltante 3,70 × costo 3.200 | `costoReposicion: 11840.00` | Positivo |
| CP-16 | — | Resumen para el indicador | `GET /alertas/resumen` | `{total, agotadas, bajas}` coherentes con el listado | Positivo |
| CP-17 | — | Sugerencia de compra | `GET /alertas/sugerencia-compra` | Agrupado por proveedor de la última compra recibida (R-06) | Positivo |
| CP-18 | — | Cantidad sugerida con margen | faltante 3,70 | `cantidadSugerida: 4.44` (+20%, R-06) | Positivo |
| CP-19 | — | MP sin compras previas | MP nunca comprada, en alerta | Aparece sin proveedor asignado, sin error | Borde |
| CP-20 | — | 🔴 **Sin alertas** | Todo por encima del mínimo | `items: []`, `total: 0` — sin error | Borde |
| CP-21 | — | Coherencia con el dashboard | Comparar con HU-24 CA-03 | Las mismas materias primas en ambos | Integridad |
| CP-22 | — | Coherencia tras una producción | Registrar consumo que deje bajo mínimo (HU-23) | La MP aparece de inmediato en las alertas | Integridad |
| CP-23 | — | ⏱ Rendimiento | 200 materias primas | < 2 s (RNF-02) | Rendimiento |
| CP-24 | — | Rol CONSULTA puede ver alertas | `GET /alertas` con token CONSULTA | `200` | Positivo |
| CP-25 | — | Rol VENTAS no accede a la sugerencia de compra | `GET .../sugerencia-compra` con token VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-26 | — | **Sin endpoints de escritura** | `POST /inventario/alertas` | `405` o `404` | Seguridad |

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

> **CP-08 es el caso que distingue una lista útil de una lista ordenada.** CA-04 pide ordenar
> por diferencia, y tomado literalmente pondría una materia prima con 40 kg disponibles y 60
> de faltante por encima de una agotada a la que le faltan 5. Pero la agotada **bloquea la
> producción hoy**; la otra todavía permite trabajar. Por eso el nivel manda sobre el
> faltante (R-03), y el orden por diferencia se aplica dentro de cada nivel.
>
> **CP-14 y CP-22 son pruebas de coherencia del sistema completo.** Si esta pantalla no
> muestra alertas cuando debería, el defecto rara vez está aquí: suele ser una materia prima
> sin fila de inventario (HU-08 R-01) o compras que no están sumando (§C-03).

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-01 a CP-09: la consulta completa con su orden de urgencia
SELECT nombre, unidad, cantidad_disponible, stock_minimo, faltante, estado_stock
FROM v_inventario_materia_prima
WHERE estado_stock IN ('BAJO','AGOTADO') AND estado = 1
ORDER BY FIELD(estado_stock, 'AGOTADO', 'BAJO'), faltante DESC;
-- Las AGOTADO deben aparecer primero aunque su faltante sea menor (R-03)

-- 🔴 CP-02 y CP-03: la comparación es estricta
SELECT nombre, cantidad_disponible, stock_minimo, estado_stock,
       cantidad_disponible < stock_minimo AS deberia_alertar
FROM v_inventario_materia_prima
WHERE cantidad_disponible = stock_minimo;
-- deberia_alertar = 0 : con disponible igual al mínimo NO hay alerta

-- CP-10, CP-11: el caso de stock_minimo = 0
SELECT nombre, cantidad_disponible, stock_minimo, estado_stock
FROM v_inventario_materia_prima WHERE stock_minimo = 0;
-- Con disponible > 0 → 'OK'. Con disponible = 0 → 'AGOTADO'.

-- 🔴 CP-14: materias primas sin fila de inventario (defecto de HU-08 R-01)
SELECT mp.nombre FROM materias_primas mp
LEFT JOIN inventario i USING (id_materia_prima)
WHERE i.id_inventario IS NULL;
-- DEBE devolver 0 filas. Si devuelve alguna, esa materia prima aparecerá
-- siempre como AGOTADA aunque tenga existencias físicas.

-- CP-17: el proveedor de la última compra recibida (R-06)
SELECT dc.id_materia_prima, p.nombre AS proveedor, MAX(c.fecha) AS ultima_compra
FROM detalle_compra dc
  JOIN compras c USING (id_compra)
  JOIN proveedores p USING (id_proveedor)
WHERE c.estado = 'RECIBIDA'
GROUP BY dc.id_materia_prima, p.id_proveedor, p.nombre;

-- 🔴 CP-21, CP-22: coherencia con el resto del sistema
-- El conteo de esta historia debe coincidir con la tarjeta del dashboard (HU-24 CA-03)
SELECT COUNT(*) AS en_alerta,
       SUM(estado_stock = 'AGOTADO') AS agotadas
FROM v_inventario_materia_prima
WHERE estado_stock IN ('BAJO','AGOTADO') AND estado = 1;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
