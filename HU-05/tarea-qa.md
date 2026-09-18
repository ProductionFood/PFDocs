# HU-05 · Gestión de clientes — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01, CA-03 | Crear cliente con solo el nombre | `{"nombre":"Tienda La Esquina"}` | `201`, `activo: true`, contacto y teléfono `null` | Positivo |
| CP-02 | CA-01 | Nombre vacío | `{"nombre":""}` | `400 VALIDACION_FALLIDA` en `nombre` | Negativo |
| CP-03 | CA-01 | Nombre de 101 caracteres | `"A"×101` | `400` | Borde |
| CP-04 | CA-01 | Nombre de exactamente 100 | `"A"×100` | `201` | Borde |
| CP-05 | CA-02 | Crear con todos los campos | nombre + contacto + teléfono | `201` con los tres valores | Positivo |
| CP-06 | CA-02 | Contacto de 101 caracteres | `contacto` largo | `400` | Borde |
| CP-07 | CA-03 | Estado por defecto | Crear sin `estado` | `activo: true`; en base `estado = 1` | Positivo |
| CP-08 | CA-04 | Listar con paginación | `?page=0&size=5` | `200`, máx. 5, `totalElements` correcto | Positivo |
| CP-09 | CA-04 | Filtrar por nombre | `?nombre=Tien` | Solo los que empiezan por "Tien" | Positivo |
| CP-10 | CA-04 | Filtro sin coincidencias | `?nombre=zzzz` | `content: []`, `totalElements: 0` | Borde |
| CP-11 | CA-04 | Comodín en el filtro | `?nombre=%` | No devuelve todo: se trata literalmente | Seguridad |
| CP-12 | CA-04 | Orden no permitido | `?sort=estado,asc` (fuera de lista blanca) | `400 CAMPO_ORDEN_INVALIDO` | Seguridad |
| CP-13 | CA-04 | `size` excesivo | `?size=1000` | `400` | Borde |
| CP-14 | — | Editar un cliente | `PUT` con datos nuevos | `200` con los cambios | Positivo |
| CP-15 | — | Editar inexistente | `PUT /clientes/9999` | `404 RECURSO_NO_ENCONTRADO` | Negativo |
| CP-16 | CA-05 | Desactivar | `PATCH .../estado {"activo":false}` | `200`, `estado = 0`, la fila permanece | Positivo |
| CP-17 | CA-05 | **No existe DELETE** | `DELETE /clientes/7` | `405` o `404` | Negativo |
| CP-18 | — | Inyección SQL en el filtro | `?nombre=' OR 1=1 -- | Sin error 500; resultado coherente | Seguridad |
| CP-19 | — | Rol CONSULTA intenta crear | `POST` con token CONSULTA | `403 SIN_PERMISO` | Seguridad |
| CP-20 | — | Rol CONSULTA puede listar | `GET` con token CONSULTA | `200` | Positivo |

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


---

## 3. Verificación en base de datos

```sql
-- CP-07: el DEFAULT de estado
SELECT id_cliente, nombre, estado FROM clientes ORDER BY id_cliente DESC LIMIT 1;

-- CP-16 y CP-17: desactivación lógica, nunca borrado
SELECT COUNT(*) FROM clientes;              -- comparar antes/después
SELECT estado FROM clientes WHERE id_cliente = 7;   -- 0 tras desactivar

-- CP-18: la tabla sigue intacta
SELECT COUNT(*) FROM clientes;

-- Verificar que el índice de V2 se usa en el filtro por nombre
EXPLAIN SELECT id_cliente, nombre FROM clientes WHERE nombre LIKE 'Tien%';
-- Se espera: key = idx_clientes_nombre
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
