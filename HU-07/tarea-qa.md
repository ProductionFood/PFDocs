# HU-07 · Unidades de medida — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-03 | Listar todas | `GET /unidades-medida` | `200` con array plano; incluye las 9 de la semilla | Positivo |
| CP-02 | CA-03 | La respuesta NO es paginada | Revisar el JSON de CP-01 | Es un array, sin `content` ni `totalElements` | Positivo |
| CP-03 | CA-03 | Orden alfabético | Revisar CP-01 | Ordenado por nombre ascendente | Positivo |
| CP-04 | CA-01 | Crear unidad válida | `{"nombre":"Media docena","abreviatura":"1/2doc"}` | `201` con `idUnidad` | Positivo |
| CP-05 | CA-01 | Nombre vacío | `{"nombre":"","abreviatura":"x"}` | `400 VALIDACION_FALLIDA` | Negativo |
| CP-06 | CA-01 | Abreviatura vacía | `{"nombre":"Caja","abreviatura":""}` | `400` | Negativo |
| CP-07 | CA-02 | Abreviatura duplicada | `{"nombre":"Kilo","abreviatura":"kg"}` | `409 ABREVIATURA_DUPLICADA` | Negativo |
| CP-08 | CA-02 | Duplicada con otra capitalización | `{"abreviatura":"KG"}` existiendo `kg` | `409` — la collation es case-insensitive (R-02) | Borde |
| CP-09 | CA-02 | Duplicada con espacios alrededor | `{"abreviatura":"  kg  "}` | `409` tras recortar | Borde |
| CP-10 | — | Editar sin cambiar la abreviatura | `PUT` con la misma abreviatura | `200` — no debe dar duplicado contra sí misma | Borde |
| CP-11 | — | Editar el nombre de una unidad en uso | Cambiar solo `nombre` | `200` | Positivo |
| CP-12 | — | Cambiar la abreviatura de una unidad en uso | `PUT` sin `confirmar` | `409 UNIDAD_EN_USO` con el conteo de referencias | Negativo |
| CP-13 | — | **No existe DELETE** | `DELETE /unidades-medida/1` | `405` o `404` (R-03) | Negativo |
| CP-14 | — | Rol PRODUCCION puede listar | `GET` con token PRODUCCION | `200` | Positivo |
| CP-15 | — | Rol PRODUCCION no puede crear | `POST` con token PRODUCCION | `403 SIN_PERMISO` | Seguridad |
| CP-16 | — | Rol CONSULTA puede listar | `GET` con token CONSULTA | `200` | Positivo |
| CP-17 | — | Unidad inexistente | `GET /unidades-medida/9999` | `404` | Negativo |

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

> **CP-08 sorprende si no se conoce la collation.** `utf8mb4_0900_ai_ci` es
> *accent-insensitive* y *case-insensitive*, así que el `UNIQUE` trata `kg` y `KG` como la
> misma abreviatura. El comportamiento es el deseado —tener ambas sería confuso— pero el
> mensaje de error debe explicarlo, porque en pantalla los dos valores se ven distintos.

---

## 3. Verificación en base de datos

```sql
-- CP-01: la semilla V3 debe haber dejado 9 unidades
SELECT COUNT(*) FROM unidades_medida;   -- 9 (más las creadas en pruebas)

-- CP-08: el UNIQUE con collation _ai_ci es case-insensitive
SHOW CREATE TABLE unidades_medida\G
-- Se espera: UNIQUE KEY `abreviatura` (`abreviatura`)
SELECT 'kg' = 'KG' COLLATE utf8mb4_0900_ai_ci;   -- devuelve 1

-- CP-12: contar las referencias antes de editar
SELECT
  (SELECT COUNT(*) FROM materias_primas WHERE id_unidad = 1) AS en_materias,
  (SELECT COUNT(*) FROM productos       WHERE id_unidad = 1) AS en_productos,
  (SELECT COUNT(*) FROM recetas         WHERE id_unidad = 1) AS en_recetas;

-- CP-13: ninguna unidad debe desaparecer
SELECT COUNT(*) FROM unidades_medida;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
