# HU-22 · Registro de consumo de materia prima — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..05 | Registrar consumo válido | `{"idMateriaPrima":5,"cantidadReal":16.200}` | `201` con teórica, real, eficiencia y desviación | Positivo |
| CP-02 | CA-03 | 🔴 **La teórica la calcula el servidor** | Enviar `{"cantidadTeorica":999}` en el cuerpo | Se ignora; se usa la de la receta (R-01) | Seguridad |
| CP-03 | CA-03 | 🔴 **Cálculo de la teórica** | Receta rinde 30 con 5 kg; planificado 90 | `cantidadTeorica: 15.000` | Integridad |
| CP-04 | CA-04 | Cantidad real omitida | Sin `cantidadReal` | `400 VALIDACION_FALLIDA` | Negativo |
| CP-05 | CA-04 | 🔴 **`cantidad_real` no admite NULL** | `INSERT` directo con `NULL` | La base lo rechaza — `NOT NULL` desde V2 §C-08 | Integridad |
| CP-06 | CA-04 | Cantidad real negativa | `{"cantidadReal":-5}` | `400 CANTIDAD_INVALIDA` | Negativo |
| CP-07 | CA-05 | La fecha se asigna sola | Crear sin fecha | `fechaRegistro` con la hora del servidor | Positivo |
| CP-08 | CA-06 | 🔴 **Eficiencia normal** | teórica 15, real 16,2 | `eficiencia: 92.59` | Positivo |
| CP-09 | CA-06 | 🔴 **División por cero** | teórica 15, real **0** | `eficiencia: null`, **no** error ni `Infinity` (R-03) | Integridad |
| CP-10 | CA-06 | Teórica cero, real positiva | teórica 0, real 5 | `eficiencia: 0`, `desviacion: null` (R-08) | Borde |
| CP-11 | CA-06 | Ambas cero | teórica 0, real 0 | `eficiencia: null`, `desviacion: null` | Borde |
| CP-12 | CA-06 | 🔴 **Eficiencia mayor que 100%** | teórica 15, real 7,5 | `eficiencia: 200.00`, `desviacion: -50.00` (R-04) | Borde |
| CP-13 | CA-06 | Cálculo de la desviación | teórica 15, real 16,2 | `desviacionPorcentaje: 8.00` | Positivo |
| CP-14 | CA-01 | Plan no EN_PROCESO | Registrar con plan `PLANIFICADO` | `409 PLAN_NO_EN_PROCESO` | Negativo |
| CP-15 | CA-02 | Materia prima inexistente | `{"idMateriaPrima":9999}` | `409 MATERIA_PRIMA_INEXISTENTE` | Negativo |
| CP-16 | — | 🔴 **Consumo duplicado** | Registrar dos veces el mismo ingrediente | `409 CONSUMO_DUPLICADO` (R-06) | Negativo |
| CP-17 | — | 🔴 **Duplicado por SQL directo** | `INSERT` repitiendo `(id_detalle_plan, id_materia_prima)` | La base lo rechaza — `uk_consumo_mp` | Integridad |
| CP-18 | — | Ingrediente fuera de receta | Registrar una MP que no está en la receta | `201`, `cantidadTeorica: 0.000`, `fueraDeReceta: true` (R-08) | Borde |
| CP-19 | — | 🔴 **El inventario se descuenta** (HU-23) | Tras CP-01 | El inventario baja 16,20 kg | Integridad |
| CP-20 | — | 🔴 **Redondeo de 3 a 2 decimales** | `cantidadReal: 16.205` | El inventario baja `16.21` (HALF_UP, R-09) | Integridad |
| CP-21 | — | Stock insuficiente | Consumir más de lo disponible | `409 STOCK_INSUFICIENTE`; **no se registra el consumo** | Negativo |
| CP-22 | — | 🔴 **Corregir ajusta por diferencia** | De 15 a 18 kg | Se descuentan **3** más, no 18 (R-07) | Integridad |
| CP-23 | — | Corregir hacia abajo devuelve | De 15 a 12 kg | Se **devuelven 3** al inventario | Integridad |
| CP-24 | — | Rol VENTAS no puede registrar consumos | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |

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

> **CP-02, CP-09 y CP-20 son los casos críticos.**
>
> **CP-02:** si el DTO acepta `cantidadTeorica`, cualquiera puede editar el JSON y fabricar
> una eficiencia del 100%. El reporte de HU-26 deja de medir nada.
>
> **CP-09:** la división por cero no lanza error en MySQL, devuelve `NULL`. El reporte se
> llena de huecos y la causa es invisible desde la interfaz.
>
> **CP-20:** el consumo tiene 3 decimales y el inventario 2. Un redondeo inconsistente
> acumula gramos en cada producción, y a los dos meses el inventario no cuadra con la bodega
> sin que ningún movimiento individual parezca incorrecto.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-09 a CP-12: la división por cero de §C-09
SELECT c.id_consumo, c.cantidad_teorica, c.cantidad_real,
       CASE WHEN c.cantidad_real = 0 THEN NULL
            ELSE ROUND(c.cantidad_teorica / c.cantidad_real * 100, 2) END AS eficiencia,
       CASE WHEN c.cantidad_teorica = 0 THEN NULL
            ELSE ROUND((c.cantidad_real - c.cantidad_teorica) / c.cantidad_teorica * 100, 2)
       END AS desviacion
FROM consumo_materia_prima c WHERE c.id_detalle_plan = 21;
-- Sin el CASE, MySQL devuelve NULL en silencio al dividir por cero.
-- Verificar que la API devuelve null y no 0, Infinity o un error.

-- 🔴 CP-03: la teórica debe coincidir con la receta
SELECT dpp.cantidad_planificada, r.cantidad_producir, dr.cantidad AS por_tanda,
       ROUND(dpp.cantidad_planificada / r.cantidad_producir * dr.cantidad, 3) AS teorica_esperada,
       c.cantidad_teorica AS teorica_registrada
FROM consumo_materia_prima c
  JOIN detalle_plan_produccion dpp USING (id_detalle_plan)
  JOIN recetas r ON r.id_producto = dpp.id_producto
  JOIN detalle_receta dr ON dr.id_receta = r.id_receta
                        AND dr.id_materia_prima = c.id_materia_prima
WHERE c.id_consumo = 31;
-- teorica_esperada y teorica_registrada DEBEN coincidir

-- 🔴 CP-05: la corrección de V2 §C-08
SHOW CREATE TABLE consumo_materia_prima\G
-- `cantidad_real` decimal(10,3) NOT NULL
-- UNIQUE KEY `uk_consumo_mp` (`id_detalle_plan`,`id_materia_prima`)

-- 🔴 CP-19 y CP-20: el descuento y su redondeo
SELECT tipo_movimiento, cantidad, saldo_anterior, saldo_posterior
FROM movimientos_inventario_mp
WHERE tabla_origen = 'consumo_materia_prima' ORDER BY id_movimiento DESC LIMIT 3;
-- SALIDA_PRODUCCION. Con cantidadReal = 16.205, el descuento debe ser 16.21

-- 🔴 CP-22: el ajuste por diferencia deja UN movimiento
SELECT tipo_movimiento, cantidad, fecha FROM movimientos_inventario_mp
WHERE tabla_origen = 'consumo_materia_prima' AND id_origen = 31
ORDER BY id_movimiento;
-- Al corregir de 15 a 18: un SALIDA_PRODUCCION de 3.
-- Si aparece ENTRADA de 15 + SALIDA de 18, se implementó revertir-y-redescontar.

-- Conciliación (05-ESTANDARES-QA.md §5): 0 filas
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
