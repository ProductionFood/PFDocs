# HU-03 · Inicio de sesión — Tarea Backend

El corazón de la seguridad del sistema. **Bloquea todas las historias posteriores**: sin login no hay forma de probar ningún endpoint protegido. La configuración completa para Spring Security 7 está en `00-base/06-ARQUITECTURA-BACKEND.md` §2 — los tutoriales de Spring Boot 3.x que se encuentran en línea no compilan aquí.

---

## 1. Pasos

1. `JwtService`: `generar(Usuario)` y `validarYExtraer(token)` con jjwt 0.13
   (API fluida nueva: `subject()`, `parser()`, `signWith(key)`).
2. `SecurityConfig` completo: `csrf` deshabilitado, `STATELESS`, `permitAll` en
   `/auth/login` y Swagger, `@EnableMethodSecurity`, `PasswordEncoder`.
3. `JwtAuthenticationFilter extends OncePerRequestFilter`:
   - extrae `Bearer`, valida, construye la autenticación con authority `ROLE_<rol>`;
   - **consulta `estado` del usuario** (R-05) → si es 0, no autentica.
4. `UsuarioAutenticado` (record) como *principal*: `idUsuario`, `correo`, `rol`.
5. `SecurityUtils.idUsuarioActual()` — lo usan HU-04 y el kardex.
6. `AuthenticationEntryPoint` y `AccessDeniedHandler` que devuelven el `ErrorResponse` del
   contrato. Sin ellos, Spring responde HTML o un JSON con otro formato y el frontend no
   puede leer `code`.
7. `AuthService.login()` respetando el orden de R-02 y el `matches()` ficticio de R-03.
8. `AuthController`: `POST /login` (público) y `GET /perfil`.
9. Registrar en bitácora los intentos fallidos con `id_usuario = NULL`
   (posible gracias a §C-11).
   > **Diferido a HU-04.** No se implementó en esta historia. HU-04 (Bitácora de
   > auditoría) depende de HU-03 y es la que construye `BitacoraService`;
   > hacerlo aquí habría obligated a montar el servicio por adelantado y volver a
   > tocarlo después. El registro de `LOGIN_FALLIDO` se cierra en HU-04.
10. Configurar `app.jwt.secret` desde variable de entorno.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/
├── security/
│   ├── SecurityConfig.java
│   ├── JwtService.java
│   ├── JwtAuthenticationFilter.java
│   ├── UsuarioAutenticado.java
│   ├── SecurityUtils.java
│   ├── ApiAuthenticationEntryPoint.java
│   └── ApiAccessDeniedHandler.java
└── auth/
    ├── {AuthController,AuthService}.java
    └── dto/{LoginRequest,LoginResponse,PerfilResponse}.java
```

---

## 3. Puntos de cuidado

- 🔒 **`JWT_SECRET` nunca se versiona.** Quien lo tenga puede firmar tokens con el rol que
  quiera. Generar con `openssl rand -base64 48`; mínimo 256 bits o jjwt lanza
  `WeakKeyException`.
- **`@EnableMethodSecurity`**: sin ella, todas las `@PreAuthorize` del proyecto se ignoran
  y la aplicación parece funcionar perfectamente.
- **Prefijo `ROLE_`**: `hasRole('ADMIN')` busca la authority `ROLE_ADMIN`. Es la causa más
  común de un `403` inexplicable.
- **Mismo mensaje para correo inexistente y contraseña errónea** (R-01), y el `matches()`
  ficticio para igualar tiempos (R-03).
- El orden de R-02 no es un detalle de estilo: invertirlo filtra qué correos existen.
- **La respuesta de login no incluye el hash** de la contraseña.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [x] Operaciones de escritura auditadas en bitácora — *diferido a HU-04, ver paso 9.*
- [x] Pruebas unitarias de las reglas con ramificación.
- [x] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.

---

## 5. Estado de ejecución (2026-10-12)

**El login ya existía**: se entregó como parte de HU-01
(`feat(HU-01): AuthController + AuthService para login con JWT`). Esta historia
no construye desde cero, **alinea** lo existente con el contrato y completa lo que
faltaba. Los pasos 1 a 6 y el 10 ya estaban cubiertos.

### Desviaciones que había que corregir

| Regla | Estado al empezar | Corrección |
|---|---|---|
| CA-04 | Credenciales incorrectas respondían **409** (`ConflictoNegocioException` es `CONFLICT`) | `AutenticacionException` con el status explícito → **401** |
| CA-03 / R-02 | `!estado \|\| !matches(...)` en un solo `if`: usuario inactivo nunca recibía 403 | Dos comprobaciones secuenciales: contraseña, después estado |
| R-03 | Sin `matches()` si el correo no existía: fuga por tiempo de respuesta | `matches()` contra un hash ficticio, generado una vez en el constructor |
| §2 / CP-16 | `GET /auth/perfil` **no existía** | `PerfilResponse` + `SecurityUtils.usuarioActual()` |
| §4 | Respuesta plana `{token,tipo,expiresIn,correo,nombre,rol}` | Anidada `{token,tipo,expiraEn,usuario{...}}`; `expiraEn` derivado de `app.jwt.expiration-ms` |

Además, fuera de los criterios de aceptación pero alcanzables desde una petición
con cabecera `Bearer`, se corrigieron dos defectos del filtro: un `sub` no numérico
lanzaba `NumberFormatException` (que **no** es `JwtException` y no lo capturaba el
`catch`, así que terminaba en **500** en vez de 401), y la condición
`usuario.isEmpty() || !estado` respondía 403 `USUARIO_INACTIVO` a un usuario
inexistente.

### Dos cosas que conviene saber para las historias siguientes

1. **El login no lleva `@PreAuthorize`**: es público. Si alguna HU añade una ruta
   bajo `/api/v1/auth/**`, hay que revisar `RutasPublicas` y `SecurityConfig` a la
   vez, porque el filtro corre **antes** que las reglas de autorización.
2. **`SecurityUtils` gana `usuarioActual()`**, que devuelve el principal completo.
   HU-04 y el kardex deberían preferirlo a encadenar los getters sueltos.

### Revisión ciega

El cambio pasó por dos revisiones adversariales ciegas en paralelo. Confirmaron un
defecto MAJOR que ninguna prueba previa detectaba: el filtro **cortaba el propio
endpoint público de login**, de modo que un cliente con un token caducado no podía
volver a iniciar sesión. Era consecuencia de que el filtro y `SecurityConfig`
nombraran la ruta pública por separado; ahora comparten `RutasPublicas.LOGIN`.

### Pendiente de verificación manual

La colección Postman de la historia está escrita pero **no se ejecutó** (requiere el
backend levantado). Los casos CP-01 a CP-16 y SEC-01 a SEC-05 siguen siendo paso de
QA.
