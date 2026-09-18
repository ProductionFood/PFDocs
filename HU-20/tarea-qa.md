# HU-20 · Plan de producción — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..05 | Crear plan válido | Fechas válidas + observaciones | `201`, `estado: "PLANIFICADO"` | Positivo |
| CP-02 | CA-01 | Fecha de planificación omitida | Sin `fechaPlanificacion` | `400 VALIDACION_FALLIDA` | Negativo |
| CP-03 | CA-01 | 🔴 **Producción anterior a planificación** | `prod` = ayer, `plan` = hoy | `400 FECHAS_INVALIDAS` (R-03) | Negativo |
| CP-04 | CA-01 | Ambas fechas iguales | `prod` = `plan` = hoy | `201` — permitido (R-03) | Borde |
| CP-05 | CA-01 | Producción futura | `prod` = dentro de 7 días | `201` — es lo normal | Positivo |
| CP-06 | CA-02 | Estado inicial | Crear plan | Siempre `PLANIFICADO`, aunque se envíe otro | Positivo |
| CP-07 | CA-02 | Estado inválido | `{"estado":"EN_HORNO"}` | `400` o se ignora | Negativo |
| CP-08 | CA-03 | Sin observaciones | Omitir el campo | `201`, `observaciones: null` | Positivo |
| CP-09 | CA-03 | Observaciones de 256 caracteres | `"A"×256` | `400` | Borde |
| CP-10 | CA-04 | 🔴 **El responsable es el usuario del token** | Crear sin `idUsuario` | `responsable` = usuario autenticado (R-04) | Positivo |
| CP-11 | CA-04 | 🔴 **PRODUCCION no puede asignar a otro** | Token PRODUCCION con `idUsuario` ajeno | Se ignora; queda a su nombre (R-04) | Seguridad |
| CP-12 | CA-04 | ADMIN sí puede asignar a otro | Token ADMIN con `idUsuario` ajeno | `201` con ese responsable | Positivo |
| CP-13 | CA-04 | Usuario inexistente | `{"idUsuario":9999}` como ADMIN | `409 USUARIO_INEXISTENTE` | Negativo |
| CP-14 | — | 🔴 **No se inicia un plan sin productos** | `PLANIFICADO → EN_PROCESO` sin líneas | `409 PLAN_SIN_PRODUCTOS` (R-02) | Negativo |
| CP-15 | — | Iniciar con productos | Plan con 2 líneas → `EN_PROCESO` | `200` | Positivo |
| CP-16 | — | Completar plan | `EN_PROCESO → COMPLETADO` | `200` | Positivo |
| CP-17 | — | `COMPLETADO` es terminal | `COMPLETADO → EN_PROCESO` | `409 TRANSICION_INVALIDA` (R-01) | Negativo |
| CP-18 | — | Salto de estado | `PLANIFICADO → COMPLETADO` | `409 TRANSICION_INVALIDA` | Negativo |
| CP-19 | — | 🔴 **Cancelar NO revierte consumos** | Plan con consumos registrados → `CANCELADO` | `200`; el inventario **no cambia** (R-06) | Integridad |
| CP-20 | — | Editar plan PLANIFICADO | `PUT` sobre uno planificado | `200` | Positivo |
| CP-21 | — | Editar plan EN_PROCESO | `PUT` sobre uno en proceso | `409 PLAN_NO_EDITABLE` (R-05) | Negativo |
| CP-22 | — | Rol VENTAS no puede crear planes | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-23 | — | Rol CONSULTA puede listar | `GET` con token CONSULTA | `200` | Positivo |

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

> **CP-11 es una prueba de seguridad, no solo funcional.** Si el backend acepta el
> `idUsuario` que envía el cliente sin verificar el rol, cualquier usuario de PRODUCCION
> puede crear planes a nombre de otra persona, y la trazabilidad de quién planificó qué deja
> de valer.

---

## 3. Verificación en base de datos

```sql
-- CP-10/CP-11: el responsable debe ser quien creó el plan
SELECT p.id_plan, p.fecha_produccion, p.estado, u.nombre AS responsable
FROM planes_produccion p JOIN usuarios u USING (id_usuario)
ORDER BY p.id_plan DESC LIMIT 5;

-- 🔴 CP-19: cancelar no debe tocar el inventario
-- Antes de cancelar:
SELECT id_materia_prima, cantidad_disponible FROM inventario;
-- (ejecutar PATCH .../estado {"estado":"CANCELADO"})
-- Después: idénticos. Y ningún movimiento nuevo:
SELECT COUNT(*) FROM movimientos_inventario_mp
WHERE tabla_origen = 'planes_produccion';
-- Debe ser 0: el plan nunca genera movimientos por sí mismo

-- CP-14: contar líneas antes de iniciar
SELECT COUNT(*) FROM detalle_plan_produccion WHERE id_plan = 9;

-- CP-06 a CP-18: el CHECK de estados de V2 §C-12
SHOW CREATE TABLE planes_produccion\G
-- CHECK (`estado` in ('PLANIFICADO','EN_PROCESO','COMPLETADO','CANCELADO'))
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
