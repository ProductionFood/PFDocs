# HU-02 · Gestión de usuarios — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

> **Revisión 2026-10-01.** Se añaden casos para R-02 por `PUT` (CP-19, CP-20), R-06
> (CP-21), filtros `idRol`/`activo` y parámetros de paginación (CP-22 a CP-28). CP-18
> pasa a esperar `405 METODO_NO_PERMITIDO`. La columna **PR** indica desde qué PR de
> `tarea-backend.md` §1 se puede ejecutar cada caso.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo | PR |
|---|---|---|---|---|---|---|
| CP-01 | CA-01 | Listar con paginación | `GET /usuarios?page=0&size=5` | `200`, máx. 5 elementos, `totalElements` correcto | Positivo | C |
| CP-02 | CA-01 | Segunda página | `?page=1&size=5` | Elementos distintos a los de la página 0 | Positivo | C |
| CP-03 | CA-01 | `size` fuera de rango | `?size=500` | `400 PARAMETRO_INVALIDO` (tope 100, **no** se recorta) | Borde | A |
| CP-04 | CA-01 | Campo de orden no permitido | `?sort=password,asc` | `400 CAMPO_ORDEN_INVALIDO` | Seguridad | A |
| CP-05 | CA-02 | Filtrar por nombre | `?busqueda=mar` | Solo usuarios cuyo nombre o correo empieza por "mar" | Positivo | C |
| CP-06 | CA-02 | Filtrar por correo | `?busqueda=admin@` | El administrador semilla | Positivo | C |
| CP-07 | CA-02 | Búsqueda sin resultados | `?busqueda=zzzzz` | `200`, `content: []`, `totalElements: 0` | Borde | C |
| CP-08 | CA-02 | Comodín `%` en la búsqueda | `?busqueda=%` | **No** devuelve todos: se trata como texto literal | Seguridad | C |
| CP-09 | CA-03 | Editar nombre y rol | `PUT` con datos válidos | `200`, cambios reflejados | Positivo | D |
| CP-10 | CA-03 | Guardar sin cambiar el correo | `PUT` con el correo actual | `200` — **no** debe dar `CORREO_DUPLICADO` contra sí mismo | Borde | D |
| CP-11 | CA-03 | Editar con correo de otro usuario | `PUT` con correo ajeno | `409 CORREO_DUPLICADO` | Negativo | D |
| CP-12 | CA-03 | Enviar `password` o `activo` en el `PUT` | `{"...","password":"nueva","activo":false}` | `200`; se ignoran: la contraseña y el estado no cambian | Seguridad | D |
| CP-13 | CA-04 | Desactivar un usuario | `PATCH .../estado {"activo":false}` | `200`, `estado = 0` en base | Positivo | E |
| CP-14 | CA-04 | Usuario desactivado intenta usar su token vigente | Token emitido antes de desactivar | `403 USUARIO_INACTIVO` **inmediatamente**, no al expirar | Seguridad | E |
| CP-15 | CA-04 | Administrador se desactiva a sí mismo | `PATCH` sobre el propio id | `409 AUTO_DESACTIVACION` | Negativo | E |
| CP-16 | CA-04 | Desactivar al último administrador activo | Dejar un solo ADMIN y desactivarlo (desde otro ADMIN que luego se desactiva, o vía BD) | `409 ULTIMO_ADMIN` | Negativo | E |
| CP-17 | CA-04 | Reactivar un usuario | `{"activo":true}` | `200`; el usuario puede iniciar sesión | Positivo | E |
| CP-18 | CA-05 | **No debe existir DELETE** | `DELETE /usuarios/12` | `405 METODO_NO_PERMITIDO`, formato de error estándar (nunca `500`) | Negativo | A |
| CP-19 | CA-03 | Quitar el rol ADMIN al último administrador activo | `PUT` con `idRol` ≠ ADMIN sobre el único ADMIN activo | `409 ULTIMO_ADMIN` | Negativo | D |
| CP-20 | CA-03 | Administrador cambia su propio rol | `PUT` sobre el propio id con otro `idRol` | `409 AUTO_DEGRADACION` | Negativo | D |
| CP-21 | CA-03 | El cambio de rol tiene efecto inmediato | Admin B degradado a `VENTAS` por admin A; B reutiliza su token previo en `GET /usuarios` | `403 SIN_PERMISO` **inmediatamente** | Seguridad | B + D |
| CP-22 | CA-02 | Filtrar por rol | `?idRol=4` | Solo usuarios `VENTAS` | Positivo | C |
| CP-23 | CA-02 | Filtrar por estado | `?activo=false` | Solo usuarios inactivos | Positivo | C |
| CP-24 | CA-01 | Listado sin `sort` | `GET /usuarios` | `200`, ordenado por nombre ascendente | Borde | A + C |
| CP-25 | CA-01 | `page` no numérico | `?page=abc` | `400 PARAMETRO_INVALIDO` | Negativo | A |
| CP-26 | CA-01 | Obtener usuario inexistente | `GET /usuarios/99999` | `404 RECURSO_NO_ENCONTRADO` | Negativo | C |
| CP-27 | CA-03 | Correo con mayúsculas y espacios | `PUT` con `"  Maria.G@ProductionFood.local "` | `200`, guardado como `maria.g@productionfood.local` | Borde | D |
| CP-28 | CA-04 | Login de usuario desactivado | `POST /auth/login` con credenciales correctas de un inactivo | `409 CREDENCIALES_INVALIDAS` (decisión HU-01, no revela el motivo) | Negativo | E |

---

## 2. Batería de seguridad

Obligatoria en **cada endpoint** de esta historia (`GET`, `GET/{id}`, `PUT`, `PATCH`):

| ID | Caso | Esperado |
|---|---|---|
| SEC-01 | Sin cabecera `Authorization` | `401 NO_AUTENTICADO` |
| SEC-02 | Token malformado | `401 NO_AUTENTICADO` |
| SEC-03 | Token expirado | `401 NO_AUTENTICADO` |
| SEC-04 | Token de rol **sin** permiso | `403 SIN_PERMISO` |
| SEC-05 | Token de usuario con `estado = 0` | `403 USUARIO_INACTIVO` |

> **SEC-04 es el caso crítico.** Si devuelve `200`, falta `@EnableMethodSecurity` y la
> autorización del sistema completo está desactivada.

> **CP-14, CP-16, CP-19 y CP-21 son los casos que importan.**
> CP-14 y CP-21 detectan si el backend confía en el token en lugar de la base: un usuario
> despedido o degradado conservaría su acceso hasta una hora.
> CP-16 y CP-19 protegen las **dos** vías para dejar el sistema sin administrador
> (desactivar y degradar), situación que solo se resuelve con acceso directo a la base.

---

## 3. Verificación en base de datos

```sql
-- CP-13: la desactivación es lógica, el registro permanece
SELECT id_usuario, correo, estado FROM usuarios WHERE id_usuario = 12;
-- Se espera: estado = 0, la fila existe

-- CP-16 / CP-19: cuántos administradores activos quedan
SELECT COUNT(*) FROM usuarios u JOIN roles r ON r.id_rol = u.id_rol
WHERE r.nombre = 'ADMIN' AND u.estado = 1;
-- Nunca debe poder llegar a 0 por la API

-- CP-12: la contraseña no cambió
SELECT password FROM usuarios WHERE id_usuario = 12;   -- mismo hash antes y después

-- CP-18: ningún usuario se borra jamás
SELECT COUNT(*) FROM usuarios;   -- comparar antes y después de las pruebas
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en los cuatro endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman HU-02 actualizada y versionada (`Backend/Docs/Postman/`).
- [ ] Auditoría en bitácora **no** se prueba aquí (R-07 → HU-04).
