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
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
