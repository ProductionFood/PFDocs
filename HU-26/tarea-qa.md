# HU-26 · Eficiencia de producción — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Cumplimiento por producto | Planificado 90, producido 60 | `cumplimiento: 66.67` | Positivo |
| CP-02 | CA-01 | Cumplimiento del 100% | Planificado 30, producido 30 | `100.00` | Borde |
| CP-03 | CA-01 | Producción por encima del plan | Planificado 90, producido 100 | `111.11` (HU-21 R-04) | Borde |
| CP-04 | CA-01 | Sin producción registrada | Planificado 90, producido 0 | `cumplimiento: 0.00` | Borde |
| CP-05 | CA-02 | Teórico vs. real | Teórica 15, real 16,2 | `desviacion: 8.00`, `eficiencia: 92.59` | Positivo |
| CP-06 | CA-02 | 🔴 **Rendimiento con real = 0** | Teórica 15, real 0 | `eficiencia: null`, **no** `0` ni error (R-03) | Integridad |
| CP-07 | CA-02 | 🔴 **Desviación con teórica = 0** | Ingrediente fuera de receta | `desviacion: null` (R-03) | Integridad |
| CP-08 | CA-03 | 🔴 **Los nulos no se promedian como cero** | 3 consumos: 92%, 100% y `null` | Promedio = **96%**, no 64%. `registrosSinRendimiento: 1` (R-03) | Integridad |
| CP-09 | CA-03 | 🔴 **Rendimiento ajustado** | Teórica 15 (para 90), producido 60, real 10 | `rendimientoAjustado: 100.00` (R-02) | Integridad |
| CP-10 | CA-03 | 🔴 **Ajustado vs. sin ajustar** | Mismo caso que CP-09 | `rendimiento: 150.00` pero `rendimientoAjustado: 100.00` | Integridad |
| CP-11 | CA-03 | 🔴 **Cumplimiento y rendimiento no se mezclan** | Cumplimiento 66%, rendimiento 92% | Son **dos campos distintos**, sin promedio entre sí (R-01) | Integridad |
| CP-12 | CA-04 | Filtrar por rango | `?fechaInicio=2026-09-01&fechaFin=2026-09-30` | Solo planes con producción en el rango | Positivo |
| CP-13 | CA-04 | Rango inclusivo | Planes el 01/09 y el 30/09 | Ambos incluidos | Borde |
| CP-14 | CA-04 | Rango invertido | `inicio > fin` | `400 RANGO_FECHAS_INVALIDO` | Negativo |
| CP-15 | — | Solo planes con consumo | Plan `PLANIFICADO` sin consumos | **No aparece** en el reporte (R-05) | Borde |
| CP-16 | — | Planes cancelados aparte | Período con 2 planes cancelados | `planesCancelados: 2`, fuera del promedio general | Integridad |
| CP-17 | — | 🔴 **Desviación acumulada por materia prima** | Harina: teórica 180, real 192,6 en 12 planes | `desviacion: 7.00`, `tendencia: "CONSUMO_EXCESIVO"` (R-06) | Positivo |
| CP-18 | — | Materia prima con desviación normal | Teórica 1,8 real 1,795 | `tendencia: "NORMAL"` | Borde |
| CP-19 | — | Productos sin receta listados | Plan con un producto sin receta | Aparece en `sinReceta[]`, no se omite (R-07) | Borde |
| CP-20 | — | Período sin planes | Rango vacío | Cifras en 0 o `null`, sin error | Borde |
| CP-21 | — | Detalle de un plan | `GET /reportes/eficiencia/9` | Productos y consumos del plan | Positivo |
| CP-22 | — | Plan sin consumos | `GET /reportes/eficiencia/{id}` de un plan sin consumo | `404 PLAN_SIN_CONSUMOS` | Negativo |
| CP-23 | — | ⏱ Rendimiento | 100 planes con 5 consumos cada uno | < 2 s (RNF-02) | Rendimiento |
| CP-24 | — | Rol VENTAS no accede | `GET /reportes/eficiencia` con token VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-25 | — | Rol COMPRAS accede a desviación de materias primas | `GET .../materias-primas` con token COMPRAS | `200` | Positivo |

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

> **CP-08 y CP-10 son los casos que distinguen un reporte útil de uno engañoso.**
>
> **CP-08:** si los `null` se promedian como cero, cada consumo registrado en cero hunde el
> indicador general. El reporte diría que la producción es ineficiente cuando en realidad
> hay un dato faltante.
>
> **CP-10:** sin el ajuste de R-02, el plan que produjo 60 de 90 aparece con un rendimiento
> del 150%, es decir, como el más eficiente del período. La conclusión es exactamente la
> contraria a la realidad, y quien lea el reporte premiará el plan que peor cumplió.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-06 a CP-08: la división por cero y el promedio. El núcleo de esta historia.
SELECT c.id_consumo, mp.nombre, c.cantidad_teorica, c.cantidad_real,
       CASE WHEN c.cantidad_real = 0 THEN NULL
            ELSE ROUND(c.cantidad_teorica / c.cantidad_real * 100, 2) END AS rendimiento,
       CASE WHEN c.cantidad_teorica = 0 THEN NULL
            ELSE ROUND((c.cantidad_real - c.cantidad_teorica) / c.cantidad_teorica * 100, 2)
       END AS desviacion
FROM consumo_materia_prima c
  JOIN materias_primas mp USING (id_materia_prima)
WHERE c.id_detalle_plan = 21;

-- 🔴 CP-08: AVG() de SQL ignora los NULL — el promedio debe coincidir con la API
SELECT ROUND(AVG(CASE WHEN cantidad_real = 0 THEN NULL
                      ELSE cantidad_teorica / cantidad_real * 100 END), 2) AS promedio,
       COUNT(*) AS total_registros,
       SUM(CASE WHEN cantidad_real = 0 THEN 1 ELSE 0 END) AS excluidos
FROM consumo_materia_prima;
-- Si la API devuelve un promedio MENOR que este, está tratando los NULL como cero.

-- 🔴 CP-09, CP-10: el rendimiento ajustado (R-02)
SELECT dpp.cantidad_planificada, dpp.cantidad_producida,
       c.cantidad_teorica, c.cantidad_real,
       ROUND(c.cantidad_teorica / NULLIF(c.cantidad_real,0) * 100, 2) AS rendimiento_bruto,
       ROUND((c.cantidad_teorica * dpp.cantidad_producida / dpp.cantidad_planificada)
             / NULLIF(c.cantidad_real,0) * 100, 2) AS rendimiento_ajustado
FROM consumo_materia_prima c
  JOIN detalle_plan_produccion dpp USING (id_detalle_plan)
WHERE c.id_detalle_plan = 21;
-- Con planificado 90, producido 60, teórica 15 y real 10:
--   rendimiento_bruto    = 150,00  ← sugiere un ahorro que no existe
--   rendimiento_ajustado = 100,00  ← la verdad: consumo proporcional correcto

-- 🔴 CP-17: la desviación acumulada, el dato más accionable (R-06)
SELECT mp.nombre, u.abreviatura,
       SUM(c.cantidad_teorica) AS teorica_total,
       SUM(c.cantidad_real)    AS real_total,
       ROUND((SUM(c.cantidad_real) - SUM(c.cantidad_teorica))
             / NULLIF(SUM(c.cantidad_teorica),0) * 100, 2) AS desviacion_pct,
       COUNT(DISTINCT dpp.id_plan) AS planes
FROM consumo_materia_prima c
  JOIN detalle_plan_produccion dpp USING (id_detalle_plan)
  JOIN planes_produccion pp USING (id_plan)
  JOIN materias_primas mp USING (id_materia_prima)
  JOIN unidades_medida u ON u.id_unidad = mp.id_unidad
WHERE pp.fecha_produccion BETWEEN '2026-09-01' AND '2026-09-30'
GROUP BY mp.id_materia_prima, mp.nombre, u.abreviatura
ORDER BY ABS(desviacion_pct) DESC;
-- Una desviación sostenida del +7% en harina indica receta mal calibrada
-- o desperdicio sistemático. Un plan aislado no lo revela; treinta sí.

-- CP-23: el índice de V2
EXPLAIN SELECT id_plan FROM planes_produccion
WHERE fecha_produccion BETWEEN '2026-09-01' AND '2026-09-30';
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
