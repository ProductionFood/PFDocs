# HU-04 · Bitácora de auditoría — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Crear un usuario genera registro | `POST /usuarios` y luego consultar bitácora | Existe un registro `CREAR` / `usuarios` con el id del autor | Positivo |
| CP-02 | CA-01 | Editar genera registro | `PUT /usuarios/{id}` | Registro `EDITAR` | Positivo |
| CP-03 | CA-01 | Cambiar estado genera registro | `PATCH /usuarios/{id}/estado` | Registro `CAMBIAR_ESTADO` | Positivo |
| CP-04 | CA-01 | Una consulta NO genera registro | `GET /usuarios` | **No** aparece registro nuevo | Negativo |
| CP-05 | CA-01 | Operación fallida NO genera registro | `POST /usuarios` con correo duplicado (`409`) | **No** aparece registro (R-02) | Negativo |
| CP-06 | CA-02 | Se almacenan los cinco campos | Consultar cualquier registro | `idUsuario`, `accion`, `tablaAfectada`, `detalle`, `fecha` presentes | Positivo |
| CP-07 | CA-03 | La fecha se asigna sola | Insertar sin enviar fecha | `fecha` con la hora actual del servidor | Positivo |
| CP-08 | CA-03 | La fecha está en hora de Colombia | Crear un registro y comparar | Coincide con la hora local, no UTC | Integridad |
| CP-09 | CA-04 | **No existe POST** | `POST /api/v1/bitacora` | `405` o `404` | Seguridad |
| CP-10 | CA-04 | **No existe PUT** | `PUT /api/v1/bitacora/1` | `405` o `404` | Seguridad |
| CP-11 | CA-04 | **No existe DELETE** | `DELETE /api/v1/bitacora/1` | `405` o `404` | Seguridad |
| CP-12 | — | Login fallido se audita como sistema | Login con contraseña errónea | Registro `LOGIN_FALLIDO` con `usuario: null` | Positivo |
| CP-13 | — | **La bitácora no contiene contraseñas** | Crear usuario y revisar el `detalle` | No aparece la contraseña ni el hash (R-05) | Seguridad |
| CP-14 | — | `detalle` largo no rompe la operación | Crear entidad con nombre de 100 caracteres | La operación tiene éxito; `detalle` recortado a 255 | Borde |
| CP-15 | — | Un rol distinto de ADMIN no accede | `GET /bitacora` con token de VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-16 | — | Filtro por rango de fechas | `?fechaInicio=2026-09-01&fechaFin=2026-09-18` | Solo registros del rango, ambos días incluidos | Positivo |
| CP-17 | — | Rango de fechas invertido | `?fechaInicio=2026-09-18&fechaFin=2026-09-01` | `400 RANGO_FECHAS_INVALIDO` | Negativo |

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

> **CP-13 es el caso crítico de esta historia.** Una bitácora que registra contraseñas
> concentra en una sola tabla —consultable desde la interfaz por cualquier administrador—
> exactamente lo que el hash de HU-01 existe para proteger. El defecto aparece cuando el
> resumen se construye serializando el DTO completo en lugar de seleccionar campos.
>
> **CP-05 verifica R-02**: auditar con `@Before` en lugar de `@AfterReturning` llena la
> bitácora de operaciones que fallaron, y deja de servir como registro de lo que
> efectivamente cambió.

---

## 3. Verificación en base de datos

```sql
-- CP-06: los cinco campos de CA-02
SELECT id_bitacora, id_usuario, accion, tabla_afectada, detalle, fecha
FROM bitacora ORDER BY id_bitacora DESC LIMIT 5;

-- CP-12: los registros del sistema llevan id_usuario NULL (§C-11)
SELECT COUNT(*) FROM bitacora WHERE id_usuario IS NULL;

-- CP-13: ninguna contraseña ni hash en la bitácora. DEBE devolver 0 filas.
SELECT id_bitacora, detalle FROM bitacora
WHERE detalle LIKE '%$2a$%' OR detalle LIKE '%password%' OR detalle LIKE '%token%';

-- CP-14: ningún detalle supera el límite de la columna
SELECT MAX(CHAR_LENGTH(detalle)) FROM bitacora;   -- <= 255

-- CP-11: el conteo solo debe crecer, nunca disminuir
SELECT COUNT(*) FROM bitacora;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
