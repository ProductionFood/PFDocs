# HU-01 · Registro de usuarios — Tarea Backend

Primera historia del proyecto. Además del CRUD, aquí se montan piezas transversales que el resto reutiliza: `GlobalExceptionHandler`, `PageResponse` y el `PasswordEncoder`.

---

## 1. Pasos

### Infraestructura transversal (se hace una vez, aquí)
1. `common/error/`: `ApiException`, `RecursoNoEncontradoException`, `ConflictoNegocioException`,
   `ErrorResponse`, `GlobalExceptionHandler` — formato de `04-CONTRATO-API.md` §5.
2. `common/pagination/`: `PageRequest` (con validación de `size` ≤ 100 y lista blanca de
   `sort`) y `PageResponse<T>`.
3. `SecurityConfig` con el bean `PasswordEncoder` (BCrypt) y `@EnableMethodSecurity`.

### Módulo `rol`
4. `Rol` (POJO), `RolRowMapper`, `RolRepository.buscarPorId()` y `.listarTodos()`.
5. `RolController` con `GET /api/v1/roles`.

### Módulo `usuario`
6. `Usuario` (POJO) y `UsuarioRowMapper`. **El mapper no expone `password` hacia el DTO.**
7. `UsuarioRepository`: `insertar()` con `GeneratedKeyHolder`, `existePorCorreo()`,
   `buscarPorId()`.
8. DTO: `CrearUsuarioRequest` (record con Bean Validation), `UsuarioResponse`, `RolResponse`.
9. `UsuarioService.crear()`:
   - normaliza el correo a minúsculas y recorta espacios;
   - valida que el rol exista → `ROL_INEXISTENTE`;
   - valida correo no duplicado → `CORREO_DUPLICADO`;
   - codifica la contraseña con `passwordEncoder.encode()`;
   - inserta y devuelve el `UsuarioResponse`.
10. `UsuarioController.crear()` con `@PreAuthorize("hasRole('ADMIN')")`, devuelve `201` +
    `Location`.
11. Capturar `DuplicateKeyException` en el servicio → `CORREO_DUPLICADO` (carrera).
12. Documentar en Swagger con `@Operation` y `@ApiResponses`.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/
├── common/error/{ApiException,ErrorResponse,GlobalExceptionHandler,...}.java
├── common/pagination/{PageRequest,PageResponse}.java
├── security/SecurityConfig.java
├── rol/{Rol,RolRepository,RolRowMapper,RolController}.java
└── usuario/
    ├── {Usuario,UsuarioRepository,UsuarioRowMapper,UsuarioService,UsuarioController}.java
    └── dto/{CrearUsuarioRequest,UsuarioResponse,RolResponse}.java
```

---

## 3. Puntos de cuidado

- **`UsuarioResponse` no lleva la contraseña**, ni siquiera el hash. Revisar el record antes
  del PR: es el error más fácil de cometer y el más difícil de detectar después.
- **`@EnableMethodSecurity` en `SecurityConfig`.** Sin ella todas las `@PreAuthorize` del
  proyecto se ignoran en silencio y la aplicación parece funcionar.
- Máximo 72 caracteres de contraseña: BCrypt trunca a partir de ahí (R-03).
- No confiar solo en `existePorCorreo()`: capturar también `DuplicateKeyException`.
- El DTO **no** recibe `estado`: lo pone el `DEFAULT 1` de la base (CA-05).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Tarea de cierre del proyecto:** crear administrador real y desactivar el semilla
      `admin@productionfood.local`.
