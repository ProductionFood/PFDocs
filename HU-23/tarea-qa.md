# HU-23 · Inventario automático por producción — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | 🔴 **El consumo descuenta el inventario** | Consumir 16,2 kg de un saldo de 112,50 | Saldo posterior 96,30 | Integridad |
| CP-02 | CA-01 | 🔴 **Se registra el movimiento de kardex** | Tras CP-01 | `SALIDA_PRODUCCION` con origen `consumo_materia_prima` | Integridad |
| CP-03 | CA-01 | Descuento con varios ingredientes | 3 consumos en una petición | Los 3 inventarios bajan correctamente | Integridad |
| CP-04 | CA-02 | 🔴 **`fecha_actualizacion` cambia sola** | Anotar la fecha, consumir, volver a consultar | La fecha es posterior (R-06) | Integridad |
| CP-05 | CA-02 | La fecha la pone la base, no la aplicación | Revisar `SHOW CREATE TABLE` | `ON UPDATE CURRENT_TIMESTAMP` presente | Integridad |
| CP-06 | CA-03 | 🔴 **Alerta al quedar bajo mínimo** | Consumo que deja 46,30 con mínimo 50 | `alertas[]` con `nivel: "BAJO"` y `faltante: 3.70` | Positivo |
| CP-07 | CA-03 | Alerta al agotarse | Consumo que deja el saldo en cero | `nivel: "AGOTADO"` | Borde |
| CP-08 | CA-03 | Sin alerta si queda por encima | Consumo que deja 96,30 con mínimo 50 | `alertas: []` | Positivo |
| CP-09 | CA-03 | 🔴 **La alerta NO bloquea el registro** | Consumo que deja bajo mínimo | `201` — se registra igualmente (R-04) | Integridad |
| CP-10 | CA-03 | Stock exactamente en el mínimo | Queda 50,00 con mínimo 50,00 | **Sin alerta** — no está por debajo | Borde |
| CP-11 | CA-04 | 🔴 **Rollback con varios ingredientes** | 3 consumos; el tercero sin stock | `409`; **ninguno** de los 3 descontado (R-02) | Integridad |
| CP-12 | CA-04 | 🔴 **Rollback ante fallo posterior** | Provocar error al insertar el consumo | El inventario **no** queda descontado | Integridad |
| CP-13 | CA-04 | 🔴 **El kardex no queda huérfano** | Tras CP-12 | No hay movimiento sin su consumo asociado | Integridad |
| CP-14 | — | 🔴 **Concurrencia** | 2 consumos simultáneos de la misma MP, stock para uno | Uno `201`, otro `409`. **Nunca negativo** (R-03) | Integridad |
| CP-15 | — | 🔴 **El stock nunca queda negativo** | Cualquier escenario | `SELECT ... WHERE cantidad_disponible < 0` devuelve 0 filas | Integridad |
| CP-16 | — | Stock insuficiente | Consumir 200 con 96,30 disponibles | `409 STOCK_INSUFICIENTE` con ambas cifras (R-05) | Negativo |
| CP-17 | — | 🔴 **Redondeo de 3 a 2 decimales** | `cantidadReal: 16.205` | El inventario baja `16.21` (HALF_UP, R-08) | Integridad |
| CP-18 | — | Redondeo hacia abajo | `cantidadReal: 16.204` | El inventario baja `16.20` | Borde |
| CP-19 | — | 🔴 **Redondeo acumulado** | 10 consumos de `0.005` cada uno | El total descontado es coherente, sin deriva | Integridad |
| CP-20 | — | Materia prima sin fila de inventario | MP creada sin su fila (defecto de HU-08) | `500 INVENTARIO_NO_INICIALIZADO`, mensaje claro | Negativo |
| CP-21 | — | 🔴 **Conciliación kardex ↔ saldo** | Tras toda la batería | La consulta de conciliación devuelve **0 filas** | Integridad |
| CP-22 | — | Corregir consumo ajusta por diferencia | De 15 a 18 (HU-22 R-07) | Se descuentan 3 más; un solo movimiento | Integridad |
| CP-23 | — | ⏱ Rendimiento | Registro de 10 consumos en una petición | Responde en < 2 s (RNF-02) | Rendimiento |

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

> **Esta historia es la más importante de probar de todo el proyecto.** Sus defectos no
> producen mensajes de error: producen un inventario que dice una cosa y una bodega que dice
> otra, y la diferencia se descubre semanas después contando sacos.
>
> **CP-11 (rollback)** es el caso que más se olvida. Registrar tres ingredientes y que el
> tercero falle dejando los dos primeros descontados produce un inventario imposible de
> reconstruir: el consumo no quedó registrado, así que no hay forma de saber qué se
> descontó ni por qué.
>
> **CP-14 (concurrencia)** requiere peticiones en paralelo —`ab -n 2 -c 2` o el *runner* de
> Postman con dos hilos—. Es la única forma de detectar la falta de `FOR UPDATE`.
>
> **CP-21 (conciliación)** debe ejecutarse al cerrar **cada fase**, no solo aquí. Es la
> prueba de que el kardex sigue siendo confiable a medida que más módulos tocan el
> inventario.

---

## 3. Verificación en base de datos

```sql
-- ═══ Preparación: anotar los saldos antes ═══
SELECT id_materia_prima, cantidad_disponible, fecha_actualizacion
FROM inventario WHERE id_materia_prima IN (5, 7, 9);

-- ═══ 🔴 CP-01, CP-02: descuento y kardex ═══
-- (ejecutar POST /detalles-plan/21/consumos con 16.200 kg de la MP 5)
SELECT cantidad_disponible, fecha_actualizacion FROM inventario WHERE id_materia_prima = 5;
-- 112.50 → 96.30, y la fecha debe haber cambiado sola (CP-04)

SELECT tipo_movimiento, cantidad, saldo_anterior, saldo_posterior, tabla_origen, id_origen
FROM movimientos_inventario_mp
WHERE id_materia_prima = 5 ORDER BY id_movimiento DESC LIMIT 1;
-- SALIDA_PRODUCCION | 16.200 | 112.50 | 96.30 | consumo_materia_prima | 31

-- ═══ 🔴 CP-11, CP-12: rollback ═══
-- Preparar: MP 7 con stock insuficiente a propósito.
-- Enviar 3 consumos donde el tercero excede el stock.
SELECT id_materia_prima, cantidad_disponible FROM inventario WHERE id_materia_prima IN (5,7,9);
-- DEBEN ser idénticos a los valores previos: ningún descuento aplicado.

SELECT COUNT(*) FROM consumo_materia_prima WHERE id_detalle_plan = 21;
-- Sin incrementos

-- ═══ 🔴 CP-13: el kardex no debe tener movimientos huérfanos ═══
SELECT m.id_movimiento, m.id_origen
FROM movimientos_inventario_mp m
LEFT JOIN consumo_materia_prima c ON c.id_consumo = m.id_origen
WHERE m.tabla_origen = 'consumo_materia_prima' AND c.id_consumo IS NULL;
-- DEBE devolver 0 filas

-- ═══ 🔴 CP-15: nunca negativo ═══
SELECT id_materia_prima, cantidad_disponible FROM inventario WHERE cantidad_disponible < 0;
-- DEBE devolver 0 filas. El CHECK de V2 lo impide, pero si llegara a saltar
-- significa que el FOR UPDATE no está funcionando.

-- ═══ 🔴 CP-21: LA CONCILIACIÓN. Ejecutar al cerrar cada fase. ═══
SELECT i.id_materia_prima, mp.nombre,
       i.cantidad_disponible AS saldo_inventario,
       m.saldo_posterior     AS saldo_kardex,
       i.cantidad_disponible - COALESCE(m.saldo_posterior, 0) AS diferencia
FROM inventario i
  JOIN materias_primas mp USING (id_materia_prima)
  LEFT JOIN (
    SELECT id_materia_prima, saldo_posterior,
           ROW_NUMBER() OVER (PARTITION BY id_materia_prima ORDER BY id_movimiento DESC) rn
    FROM movimientos_inventario_mp
  ) m ON m.id_materia_prima = i.id_materia_prima AND m.rn = 1
WHERE i.cantidad_disponible <> COALESCE(m.saldo_posterior, 0);
-- DEBE devolver 0 filas SIEMPRE.
-- Una sola fila significa que alguna ruta de código actualizó el saldo
-- sin registrar el movimiento: el inventario dejó de ser explicable.

-- ═══ CP-17 a CP-19: el redondeo ═══
SELECT c.cantidad_real AS registrado_3_dec, m.cantidad AS descontado_2_dec
FROM consumo_materia_prima c
  JOIN movimientos_inventario_mp m ON m.tabla_origen = 'consumo_materia_prima'
                                  AND m.id_origen = c.id_consumo
WHERE c.id_detalle_plan = 21;
-- 16.205 → 16.21 ; 16.204 → 16.20 (HALF_UP)

-- ═══ CP-20: toda MP debe tener fila de inventario (HU-08 R-01) ═══
SELECT mp.nombre FROM materias_primas mp
LEFT JOIN inventario i USING (id_materia_prima) WHERE i.id_inventario IS NULL;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
