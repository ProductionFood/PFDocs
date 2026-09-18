# HU-01 · Registro de usuarios — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01, CA-06 | Crear usuario válido | `{"nombre":"María Gómez","correo":"maria@pf.local","password":"Clave2026*","idRol":4}` | `201`, cuerpo con `idUsuario`, `activo: true`, cabecera `Location` | Positivo |
| CP-02 | CA-01 | Nombre vacío | `{"nombre":"", ...}` | `400 VALIDACION_FALLIDA`, `fieldErrors` con `field: "nombre"` | Negativo |
| CP-03 | CA-01 | Nombre de 101 caracteres | `"A"×101` | `400`, error en `nombre` | Borde |
| CP-04 | CA-01 | Nombre de exactamente 100 | `"A"×100` | `201` | Borde |
| CP-05 | CA-02 | Correo duplicado | Correo de un usuario existente | `409 CORREO_DUPLICADO` | Negativo |
| CP-06 | CA-02 | Correo con formato inválido | `"maria.pf.local"` | `400`, error en `correo` | Negativo |
| CP-07 | CA-02 | Correo duplicado con distinta capitalización | `"MARIA@pf.local"` existiendo `maria@pf.local` | `409 CORREO_DUPLICADO` (se normaliza a minúsculas) | Borde |
| CP-08 | CA-03 | **La respuesta no debe contener la contraseña** | Crear usuario válido | El JSON **no** incluye `password` ni el hash | Seguridad |
| CP-09 | CA-03 | Contraseña de 7 caracteres | `"Corta1*"` | `400` | Borde |
| CP-10 | CA-03 | La contraseña se guarda hasheada | Crear usuario y consultar la base | `password` empieza por `$2a$` y no es el texto plano | Integridad |
| CP-11 | CA-04 | Rol inexistente | `{"idRol": 999, ...}` | `409 ROL_INEXISTENTE` | Negativo |
| CP-12 | CA-05 | Estado por defecto | Crear sin enviar `estado` | `activo: true`; en base `estado = 1` | Positivo |
| CP-13 | — | Inyección SQL en el nombre | `"'; DROP TABLE usuarios; --"` | Se guarda como texto literal; la tabla sigue existiendo | Seguridad |
| CP-14 | — | Crear usuario como rol VENTAS | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-15 | — | Listar roles | `GET /api/v1/roles` con token ADMIN | `200` con los 5 roles de la semilla | Positivo |

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

> **CP-08 merece atención especial.** Es el defecto más común en este tipo de endpoint:
> el DTO de respuesta se construye copiando la entidad completa y arrastra el hash. No
> produce ningún error visible, y el hash termina en los logs del navegador y en las
> capturas de los reportes.

---

## 3. Verificación en base de datos

```sql
-- CP-10: la contraseña debe estar hasheada, nunca en texto plano
SELECT correo, password, LENGTH(password) AS largo, estado
FROM usuarios WHERE correo = 'maria@pf.local';
-- Se espera: password empieza por '$2a$', largo = 60, estado = 1

-- CP-12: el DEFAULT 1 de V2 debe estar aplicado
SHOW CREATE TABLE usuarios\G
-- Se espera: `estado` tinyint(1) NOT NULL DEFAULT '1'

-- CP-13: la tabla sigue intacta tras el intento de inyección
SELECT COUNT(*) FROM usuarios;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
