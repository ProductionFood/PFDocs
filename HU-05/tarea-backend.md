# HU-05 · Gestión de clientes — Tarea Backend

**CRUD de referencia del proyecto.** Se implementa con cuidado porque HU-06, HU-07, HU-08 y HU-11 lo copian. El patrón completo de repositorio está en `00-base/06-ARQUITECTURA-BACKEND.md` §4.

---

## 1. Pasos

1. `Cliente` (POJO) y `ClienteRowMapper` — `estado` → `boolean` con `rs.getBoolean()`.
2. `ClienteRepository`:
   - `buscar(nombre, activo, PageRequest)` con filtros opcionales y lista blanca de orden;
   - `contar(nombre, activo)` — **mismo `WHERE`**;
   - `buscarPorId()`, `insertar()` con `GeneratedKeyHolder`, `actualizar()`,
     `cambiarEstado()`;
   - `escaparLike()` para `%` y `_`.
3. DTO: `CrearClienteRequest`, `ActualizarClienteRequest`, `ClienteResponse`,
   `CambiarEstadoRequest`.
4. `ClienteService` con `@Transactional` en escrituras y `@Transactional(readOnly = true)`
   en lecturas.
5. `ClienteController` con `@PreAuthorize` según la matriz de roles y `@Operation` de Swagger.
6. Anotar con `@Auditable` las operaciones de escritura (HU-04).
7. **No implementar `DELETE`** (CA-05).

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/cliente/
├── {Cliente,ClienteRowMapper,ClienteRepository,ClienteService,ClienteController}.java
└── dto/{CrearClienteRequest,ActualizarClienteRequest,ClienteResponse}.java
```

---

## 3. Puntos de cuidado

- **`buscar()` y `contar()` con el mismo `WHERE`**, siempre.
- **`sort` desde lista blanca**; nunca concatenar el valor recibido.
- **Escapar `%` y `_`**: sin eso, buscar `%` devuelve la tabla entera.
- `estado` se expone como `boolean` en el JSON, no como `0`/`1`
  (`04-CONTRATO-API.md` §6).
- El DTO de creación **no** recibe `estado`: lo pone el `DEFAULT` (CA-03).
- Sin endpoint `DELETE`.
- Como es la plantilla: dejar el código limpio y comentado donde la decisión no sea obvia.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
