# HU-02 · Gestión de usuarios — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Listar con paginación | `GET /usuarios?page=0&size=5` | `200`, máx. 5 elementos, `totalElements` correcto | Positivo |
| CP-02 | CA-01 | Segunda página | `?page=1&size=5` | Elementos distintos a los de la página 0 | Positivo |
| CP-03 | CA-01 | `size` fuera de rango | `?size=500` | `400 PARAMETRO_INVALIDO` (tope 100) | Borde |
| CP-04 | CA-01 | Campo de orden no permitido | `?sort=password,asc` | `400 CAMPO_ORDEN_INVALIDO` | Seguridad |
| CP-05 | CA-02 | Filtrar por nombre | `?busqueda=mar` | Solo usuarios cuyo nombre o correo empieza por "mar" | Positivo |
| CP-06 | CA-02 | Filtrar por correo | `?busqueda=admin@` | El administrador semilla | Positivo |
| CP-07 | CA-02 | Búsqueda sin resultados | `?busqueda=zzzzz` | `200`, `content: []`, `totalElements: 0` | Borde |
| CP-08 | CA-02 | Comodín `%` en la búsqueda | `?busqueda=%` | **No** devuelve todos: se trata como texto literal | Seguridad |
| CP-09 | CA-03 | Editar nombre y rol | `PUT` con datos válidos | `200`, cambios reflejados | Positivo |
| CP-10 | CA-03 | Guardar sin cambiar el correo | `PUT` con el correo actual | `200` — **no** debe dar `CORREO_DUPLICADO` contra sí mismo | Borde |
| CP-11 | CA-03 | Editar con correo de otro usuario | `PUT` con correo ajeno | `409 CORREO_DUPLICADO` | Negativo |
| CP-12 | CA-03 | Enviar `password` en el `PUT` | `{"password":"nueva"}` | Se ignora; la contraseña no cambia | Seguridad |
| CP-13 | CA-04 | Desactivar un usuario | `PATCH .../estado {"activo":false}` | `200`, `estado = 0` en base | Positivo |
| CP-14 | CA-04 | Usuario desactivado intenta usar su token vigente | Token emitido antes de desactivar | `403 USUARIO_INACTIVO` **inmediatamente**, no al expirar | Seguridad |
| CP-15 | CA-04 | Administrador se desactiva a sí mismo | `PATCH` sobre el propio id | `409 AUTO_DESACTIVACION` | Negativo |
| CP-16 | CA-04 | Desactivar al último administrador activo | Dejar un solo ADMIN y desactivarlo | `409 ULTIMO_ADMIN` | Negativo |
| CP-17 | CA-04 | Reactivar un usuario | `{"activo":true}` | `200`; el usuario puede iniciar sesión | Positivo |
| CP-18 | CA-05 | **No debe existir DELETE** | `DELETE /usuarios/12` | `405 Method Not Allowed` o `404` | Negativo |

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

> **CP-14 y CP-16 son los casos que importan.**
> CP-14 detecta si la validación de `estado` se hace solo al iniciar sesión: en ese caso un
> usuario despedido conserva acceso hasta una hora después de ser desactivado.
> CP-16 protege contra dejar el sistema sin ningún administrador, situación que solo se
> resuelve con acceso directo a la base de datos.

---

## 3. Verificación en base de datos

```sql
-- CP-13: la desactivación es lógica, el registro permanece
SELECT id_usuario, correo, estado FROM usuarios WHERE id_usuario = 12;
-- Se espera: estado = 0, la fila existe

-- CP-16: verificar cuántos administradores activos quedan
SELECT COUNT(*) FROM usuarios u JOIN roles r ON r.id_rol = u.id_rol
WHERE r.nombre = 'ADMIN' AND u.estado = 1;
-- Nunca debe poder llegar a 0 por esta vía

-- CP-18: ningún usuario se borra jamás
SELECT COUNT(*) FROM usuarios;   -- comparar antes y después de las pruebas
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
