# HU-04 · Bitácora de auditoría — Tarea Backend

Componente transversal. Se implementa una vez y lo usan todas las historias posteriores. Requiere `spring-boot-starter-aop`.

---

## 1. Pasos

1. Agregar la dependencia `spring-boot-starter-aop` al `pom.xml`.
2. `@Auditable` — anotación de método con `accion()` y `tabla()`.
3. `Bitacora` (POJO), `BitacoraRowMapper` con `LEFT JOIN usuarios` para el nombre.
   **`rs.getObject("id_usuario", Long.class)`**, no `getLong()`: este último devuelve `0`
   para `NULL` y el registro del sistema aparecería atribuido al usuario 0.
4. `BitacoraRepository`: `insertar()`, `buscar(filtros, page)`, `contar(filtros)`,
   `listarAcciones()` (`SELECT DISTINCT accion`).
5. `BitacoraService.registrar(idUsuario, accion, tabla, detalle)`:
   - recorta `detalle` a 255 caracteres (R-04);
   - **envuelto en `try/catch` que solo registra en el log** (R-03).
6. `AuditoriaAspect` con `@AfterReturning` sobre `@annotation(auditable)`:
   - obtiene el usuario con `SecurityUtils.idUsuarioActual()` (puede ser `null`);
   - construye el resumen desde el valor retornado, **sin incluir campos sensibles** (R-05).
7. `BitacoraController` con **solo métodos `GET`** y `@PreAuthorize("hasRole('ADMIN')")`.
8. Filtros: `idUsuario`, `tabla`, `accion`, `fechaInicio`, `fechaFin`, paginación.
   Validar `fechaInicio <= fechaFin` → `400 RANGO_FECHAS_INVALIDO`.
9. Anotar los servicios ya escritos de HU-01 y HU-02.
10. En HU-03, registrar `LOGIN` y `LOGIN_FALLIDO` con `id_usuario = NULL` cuando
    corresponda.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/common/audit/
├── Auditable.java
├── AuditoriaAspect.java
├── BitacoraService.java
├── Bitacora.java
├── BitacoraRepository.java
├── BitacoraRowMapper.java
├── BitacoraController.java
└── dto/BitacoraResponse.java
```

---

## 3. Puntos de cuidado

- **`getObject("id_usuario", Long.class)`**, nunca `getLong()`: `getLong()` convierte
  `NULL` en `0` y el registro aparecería atribuido a un usuario inexistente.
- **Ningún endpoint de escritura** (CA-04). Si existe un `POST /bitacora`, la auditoría
  deja de ser confiable: cualquiera podría fabricar registros.
- **Recortar `detalle` a 255** antes de insertar (R-04). Con `sql_mode` estricto el `INSERT`
  falla, y por R-03 el fallo se traga en silencio.
- **Nunca auditar contraseñas ni tokens** (R-05). Revisar el método que construye el
  resumen antes del PR.
- `@AfterReturning`, no `@Before` (R-02).
- El `try/catch` de R-03 **no debe capturar y relanzar**: debe tragarse la excepción y
  registrarla en el log.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
