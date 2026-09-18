# HU-21 · Productos en plan de producción — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..04 | Agregar producto al plan | `{"idProducto":3,"cantidadPlanificada":90}` | `201`, `cantidadProducida: 0.00` | Positivo |
| CP-02 | CA-04 | 🔴 **`cantidadProducida` inicia en cero** | Tras CP-01 | `0.00` en la respuesta y en la base | Positivo |
| CP-03 | CA-01 | Plan inexistente | `POST /planes-produccion/9999/detalles` | `404 PLAN_INEXISTENTE` | Negativo |
| CP-04 | CA-01 | Agregar a plan EN_PROCESO | `POST` sobre uno en proceso | `409 PLAN_NO_EDITABLE` | Negativo |
| CP-05 | CA-02 | Producto inexistente | `{"idProducto":9999}` | `409 PRODUCTO_INEXISTENTE` | Negativo |
| CP-06 | CA-03 | Cantidad planificada cero | `{"cantidadPlanificada":0}` | `400 CANTIDAD_INVALIDA` — `CHECK > 0` | Negativo |
| CP-07 | CA-03 | Cantidad negativa | `{"cantidadPlanificada":-10}` | `400` | Negativo |
| CP-08 | CA-04 | 🔴 **Registrar producción con plan EN_PROCESO** | `PATCH .../producido {"cantidadProducida":60}` | `200`, `avance: 66.67` | Positivo |
| CP-09 | CA-04 | 🔴 **No se registra producción con plan PLANIFICADO** | `PATCH` sobre plan planificado | `409 PLAN_NO_EN_PROCESO` (R-03) | Negativo |
| CP-10 | CA-04 | No se registra producción con plan COMPLETADO | `PATCH` sobre plan completado | `409 PLAN_NO_EN_PROCESO` | Negativo |
| CP-11 | CA-04 | Producción cero | `{"cantidadProducida":0}` | `200` — válido, significa que no se produjo nada | Borde |
| CP-12 | CA-04 | 🔴 **Se puede producir más de lo planificado** | Planificado 90, producido 100 | `200`, `avance: 111.11` (R-04) | Borde |
| CP-13 | — | 🔴 **No se edita lo planificado con el plan EN_PROCESO** | `PUT .../detalles/21` en proceso | `409 PLAN_NO_EDITABLE` (R-02) | Negativo |
| CP-14 | — | Editar lo planificado con plan PLANIFICADO | `PUT` sobre plan planificado | `200` | Positivo |
| CP-15 | CA-05 | Listar el detalle | `GET /planes-produccion/9/detalles` | Líneas con avance y consumo teórico | Positivo |
| CP-16 | — | 🔴 **Producto duplicado en el plan** | Agregar uno ya presente | `409 PRODUCTO_DUPLICADO` (R-05) | Negativo |
| CP-17 | — | 🔴 **Duplicado por SQL directo** | `INSERT` repitiendo `(id_plan, id_producto)` | La base lo rechaza — `uk_detalle_plan` | Integridad |
| CP-18 | — | 🔴 **Consumo teórico correcto** | Receta rinde 30 con 5 kg; planificar 90 | Consumo teórico = **15,000 kg** (HU-12 R-02) | Integridad |
| CP-19 | — | Producto sin receta | Agregar uno sin receta | `201`, `tieneReceta: false`, sin consumo teórico (R-06) | Borde |
| CP-20 | — | 🔴 **Registrar producción NO crea lote ni stock** | Tras CP-08 | `inventario_producto_terminado` **no cambia** (R-07) | Integridad |
| CP-21 | — | Quitar línea sin consumos | `DELETE` de una línea limpia | `204` | Positivo |
| CP-22 | — | 🔴 **Quitar línea con consumos** | `DELETE` tras registrar consumo en HU-22 | `409 DETALLE_CON_CONSUMOS` (R-08) | Negativo |
| CP-23 | — | Rol VENTAS no puede agregar | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |

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

> **CP-09 y CP-13 protegen el sentido de HU-26.** Si se puede registrar producción antes de
> iniciar, o editar lo planificado después de producir, la comparación
> planificado-vs-producido deja de medir nada: siempre se podría ajustar el objetivo al
> resultado.
>
> **CP-20 documenta el hueco de R-07.** No es un defecto de esta historia: es el
> comportamiento acordado. Conviene tenerlo presente porque significa que completar un plan
> **no** pone el producto a la venta.

---

## 3. Verificación en base de datos

```sql
-- CP-02: el DEFAULT de cantidad_producida
SELECT id_detalle_plan, cantidad_planificada, cantidad_producida
FROM detalle_plan_produccion WHERE id_plan = 9;

-- 🔴 CP-17: la restricción de V2 §C-13. La segunda DEBE fallar.
INSERT INTO detalle_plan_produccion (id_plan, id_producto, cantidad_planificada)
VALUES (9, 3, 50);
INSERT INTO detalle_plan_produccion (id_plan, id_producto, cantidad_planificada)
VALUES (9, 3, 40);
-- Error 1062. Sin el UNIQUE, HU-22 calcularía el consumo teórico dos veces.

-- 🔴 CP-18: verificar el consumo teórico contra la receta
SELECT dpp.cantidad_planificada,
       r.cantidad_producir,
       mp.nombre AS ingrediente,
       dr.cantidad AS por_tanda,
       ROUND(dpp.cantidad_planificada / r.cantidad_producir * dr.cantidad, 3) AS teorico
FROM detalle_plan_produccion dpp
  JOIN recetas r        ON r.id_producto = dpp.id_producto
  JOIN detalle_receta dr USING (id_receta)
  JOIN materias_primas mp USING (id_materia_prima)
WHERE dpp.id_detalle_plan = 21;
-- 90 / 30 × 5,000 = 15,000 kg de harina.
-- Un resultado absurdo (450 kg) indica que cantidad_producir de la receta
-- se interpretó mal (HU-12 R-02).

-- 🔴 CP-20: registrar producción no debe tocar el stock de producto terminado
SELECT COUNT(*) FROM movimientos_inventario_pt
WHERE tabla_origen = 'detalle_plan_produccion';
-- Debe ser 0: el lote se registra aparte, en HU-19 (R-07)

-- CP-12: producir de más es válido
SELECT cantidad_planificada, cantidad_producida,
       ROUND(cantidad_producida / cantidad_planificada * 100, 2) AS avance
FROM detalle_plan_produccion WHERE id_plan = 9;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
