# HU-08 · Gestión de materias primas — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01, CA-04 | Crear materia prima válida | `{"nombre":"Harina de trigo","idUnidad":1,"stockMinimo":50,"costoUnitario":3200}` | `201`, `activo: true`, `cantidadDisponible: 0` | Positivo |
| CP-02 | — | **Se crea la fila de inventario** | Tras CP-01, consultar `inventario` | Existe fila con `cantidad_disponible = 0.00` (R-01) | Integridad |
| CP-03 | CA-01 | Nombre vacío | `{"nombre":""}` | `400 VALIDACION_FALLIDA` | Negativo |
| CP-04 | CA-01 | Nombre de 101 caracteres | `"A"×101` | `400` | Borde |
| CP-05 | CA-02 | Unidad inexistente | `{"idUnidad":9999}` | `409 UNIDAD_INEXISTENTE` | Negativo |
| CP-06 | CA-02 | Unidad omitida | Sin `idUnidad` | `400` | Negativo |
| CP-07 | CA-03 | Stock mínimo omitido | Sin `stockMinimo` | `400` | Negativo |
| CP-08 | CA-03 | Stock mínimo negativo | `{"stockMinimo":-5}` | `400` o `409` — `CHECK >= 0` de V2 | Negativo |
| CP-09 | CA-03 | Stock mínimo cero | `{"stockMinimo":0}` | `201` — es válido (R-06) | Borde |
| CP-10 | CA-03 | Costo con 2 decimales | `{"costoUnitario":3200.55}` | `201`; en base `3200.55` exacto | Positivo |
| CP-11 | CA-03 | Costo con 3 decimales | `{"costoUnitario":3200.555}` | Se redondea a `3200.56` (`decimal(10,2)`) | Borde |
| CP-12 | CA-04 | Estado por defecto | Crear sin `estado` | `activo: true`; en base `estado = 1` | Positivo |
| CP-13 | CA-05 | Listar paginado | `?page=0&size=5` | `200`, máx. 5 | Positivo |
| CP-14 | CA-05 | Filtrar por nombre | `?nombre=Hari` | Solo los que empiezan por "Hari" | Positivo |
| CP-15 | CA-05 | Orden por costo | `?sort=costoUnitario,desc` | Ordenado descendente | Positivo |
| CP-16 | CA-05 | Orden fuera de lista blanca | `?sort=estado,asc` | `400 CAMPO_ORDEN_INVALIDO` | Seguridad |
| CP-17 | — | Editar materia prima | `PUT` con datos nuevos | `200` | Positivo |
| CP-18 | — | Desactivar conserva el stock | Desactivar una con inventario > 0 | `200`; `cantidad_disponible` no cambia (R-04) | Integridad |
| CP-19 | — | **No existe DELETE** | `DELETE /materias-primas/5` | `405` o `404` | Negativo |
| CP-20 | — | Rol COMPRAS puede listar | `GET` con token COMPRAS | `200` | Positivo |
| CP-21 | — | Rol COMPRAS no puede crear | `POST` con token COMPRAS | `403 SIN_PERMISO` | Seguridad |
| CP-22 | — | Rol PRODUCCION no puede crear | `POST` con token PRODUCCION | `403 SIN_PERMISO` | Seguridad |

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

> **CP-02 es el caso crítico de esta historia** y conviene ejecutarlo tras cada alta durante
> todo el proyecto. Una materia prima sin fila de inventario no da ningún error al crearse:
> el defecto aparece semanas después, cuando HU-23 intenta descontar de una fila que no
> existe o HU-27 no detecta un desabastecimiento. La consulta de conciliación de arriba lo
> detecta en un segundo.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-02: el caso que más importa de esta historia.
-- Toda materia prima DEBE tener su fila de inventario (R-01).
SELECT mp.id_materia_prima, mp.nombre, i.cantidad_disponible
FROM materias_primas mp
LEFT JOIN inventario i ON i.id_materia_prima = mp.id_materia_prima
WHERE i.id_inventario IS NULL;
-- DEBE devolver 0 filas. Cualquier fila aquí es un defecto crítico:
-- esa materia prima romperá HU-10, HU-23 y HU-27.

-- CP-08: el CHECK de V2 debe estar activo
INSERT INTO materias_primas (nombre, id_unidad, stock_minimo, costo_unitario)
VALUES ('Prueba', 1, -5, 100);
-- DEBE fallar con error 3819

-- CP-12: el DEFAULT 1 añadido en V2 §C-06
SHOW CREATE TABLE materias_primas\G
-- Se espera: `estado` tinyint(1) NOT NULL DEFAULT '1'

-- CP-18: desactivar no toca el inventario
SELECT mp.estado, i.cantidad_disponible
FROM materias_primas mp JOIN inventario i USING (id_materia_prima)
WHERE mp.id_materia_prima = 5;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
