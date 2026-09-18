# HU-02 · Gestión de usuarios — Tarea Backend

Amplía el módulo `usuario` de HU-01 con listado paginado, edición y cambio de estado. Aquí se estrena el patrón de listado con filtro que replicarán todos los CRUD posteriores.

---

## 1. Pasos

1. `UsuarioRepository.buscar(busqueda, idRol, activo, PageRequest)`:
   - `WHERE 1=1` + condiciones opcionales;
   - `(nombre LIKE :q OR correo LIKE :q)` con `escaparLike(q) + "%"`;
   - `ORDER BY` desde la **lista blanca** `{nombre, correo, id}`;
   - `LIMIT :limit OFFSET :offset`.
2. `UsuarioRepository.contar(...)` — **mismo `WHERE` que `buscar()`**. Si divergen,
   `totalElements` miente y la última página aparece vacía.
3. `UsuarioRepository.actualizar()` y `.cambiarEstado()`.
4. `UsuarioRepository.existePorCorreoExcluyendo(correo, idUsuario)` — R-04.
5. `UsuarioRepository.contarAdminsActivos()` — R-02.
6. DTO: `ActualizarUsuarioRequest`, `CambiarEstadoRequest`.
7. `UsuarioService.listar()` → arma el `PageResponse<T>`.
8. `UsuarioService.actualizar()`: valida rol, valida correo único excluyendo al propio.
9. `UsuarioService.cambiarEstado()`:
   - `id == SecurityUtils.idUsuarioActual()` → `AUTO_DESACTIVACION`;
   - desactivando un `ADMIN` y `contarAdminsActivos() <= 1` → `ULTIMO_ADMIN`.
10. Controlador con `@PreAuthorize("hasRole('ADMIN')")` en los cuatro métodos.
11. Auditar `EDITAR` y `CAMBIAR_ESTADO` en bitácora (HU-04).
12. **Verificar que el filtro `estado` se consulta en cada petición** (R-05): el
    `JwtAuthenticationFilter` o el `UserDetailsService` debe rechazar usuarios inactivos.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/usuario/
├── UsuarioRepository.java      (+ buscar, contar, actualizar, cambiarEstado)
├── UsuarioService.java         (+ listar, actualizar, cambiarEstado)
├── UsuarioController.java      (+ GET, GET/{id}, PUT, PATCH)
└── dto/{ActualizarUsuarioRequest,CambiarEstadoRequest}.java
```

---

## 3. Puntos de cuidado

- **`buscar()` y `contar()` deben compartir exactamente el mismo `WHERE`.** Es el error
  más común de la paginación manual.
- **`sort` nunca se concatena crudo.** Lista blanca obligatoria.
- **Escapar `%` y `_`** en el término de búsqueda: sin eso, buscar `%` devuelve todo.
- R-02 no es hipotético: desactivar al último administrador deja el sistema sin acceso y
  solo se arregla con un `UPDATE` directo en la base.
- `PUT` no acepta `password` (R-03).
- El `LIMIT/OFFSET` va parametrizado, no interpolado.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
