# HU-03 · Inicio de sesión — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01, CA-02 | Login válido | `{"correo":"admin@productionfood.local","password":"Admin123*"}` | `200` con `token`, `tipo: "Bearer"`, datos del usuario | Positivo |
| CP-02 | CA-05 | El token contiene los claims exigidos | Decodificar el token de CP-01 en jwt.io | Contiene `sub`, `correo`, `nombre`, `rol`, `exp` | Positivo |
| CP-03 | CA-04 | Contraseña incorrecta | Correo válido + contraseña errónea | `401 CREDENCIALES_INVALIDAS` | Negativo |
| CP-04 | CA-04 | Correo inexistente | `"noexiste@x.local"` | `401` con **el mismo mensaje** que CP-03 | Seguridad |
| CP-05 | CA-04 | **Los mensajes de CP-03 y CP-04 deben ser idénticos** | Comparar ambas respuestas | `code` y `message` exactamente iguales | Seguridad |
| CP-06 | CA-04 | Tiempos de respuesta comparables | Medir CP-03 y CP-04 10 veces | Diferencia no sistemática (R-03) | Seguridad |
| CP-07 | CA-03 | Usuario desactivado | Desactivar un usuario e intentar login con credenciales correctas | `403 USUARIO_INACTIVO` | Negativo |
| CP-08 | CA-03 | Usuario desactivado con contraseña incorrecta | Desactivado + contraseña errónea | `401`, **no** `403` (R-02: no revela que existe) | Seguridad |
| CP-09 | CA-01 | Correo vacío | `{"correo":"","password":"x"}` | `400 VALIDACION_FALLIDA` | Negativo |
| CP-10 | CA-02 | El token funciona | Usar el token en `GET /usuarios` | `200` | Positivo |
| CP-11 | — | Token manipulado | Alterar un carácter de la firma | `401 NO_AUTENTICADO` | Seguridad |
| CP-12 | — | Token con rol alterado | Cambiar `"rol":"CONSULTA"` por `"ADMIN"` en el payload | `401` — la firma no valida | Seguridad |
| CP-13 | — | Token expirado | Token con `exp` vencido | `401 NO_AUTENTICADO` | Seguridad |
| CP-14 | — | La respuesta no expone el hash | Revisar el JSON de CP-01 | No aparece `password` en ningún nivel | Seguridad |
| CP-15 | — | Inyección SQL en el correo | `admin@x.local' OR '1'='1` | `401`, sin error 500 | Seguridad |
| CP-16 | — | Perfil del usuario autenticado | `GET /auth/perfil` con token | `200` con los datos del token | Positivo |

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

> **CP-05, CP-08 y CP-12 son los casos que de verdad importan en esta historia.**
>
> CP-05 y CP-08 verifican que el sistema no permita **enumerar correos registrados**: si el
> mensaje o el código HTTP cambia según si la cuenta existe, cualquiera puede averiguar
> quién tiene usuario en el sistema probando direcciones.
>
> CP-12 verifica que la firma del token se valide de verdad. Un backend que decodifica el
> JWT sin verificar la firma acepta cualquier rol que el atacante escriba — y eso otorga
> acceso de administrador a cualquiera que sepa editar un JSON en base64.

---

## 3. Verificación en base de datos

```sql
-- CP-07: verificar el estado antes de la prueba
UPDATE usuarios SET estado = 0 WHERE correo = 'prueba@productionfood.local';

-- Los intentos fallidos se registran con id_usuario NULL (§C-11)
SELECT id_usuario, accion, tabla_afectada, detalle, fecha
FROM bitacora WHERE accion = 'LOGIN_FALLIDO' ORDER BY fecha DESC LIMIT 5;

-- Restaurar tras la prueba
UPDATE usuarios SET estado = 1 WHERE correo = 'prueba@productionfood.local';
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
