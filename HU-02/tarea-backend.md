# HU-02 · Gestión de usuarios — Tarea Backend

Amplía el módulo `usuario` de HU-01 con listado paginado, edición y cambio de estado. Aquí
se estrena el patrón de listado con filtro que replicarán todos los CRUD posteriores.

> **Revisión 2026-10-01.** Reescrita sobre lo que existe tras HU-01: **Spring Data JPA**
> (`UsuarioRepository extends JpaRepository`), no JdbcTemplate. El desfase con
> `00-base/01-CONVENCIONES.md` y `06-ARQUITECTURA-BACKEND.md` sigue pendiente en el repo
> `docs`; para el código manda lo que ya está en `main`.

---

## 1. Plan de trabajo

Estimación total: **~930 líneas** (código + pruebas). Supera la referencia de 500 por
tarea, así que se parte en **tres tareas** por costuras de comportamiento. Cada una se
mergea y despliega sola sin romper nada.

| Tarea | Qué habilita | PRs | Líneas aprox. |
|---|---|---|---|
| **T1 · Cimientos** | Errores 400/405 correctos, paginación usable, permisos al día (R-06) | PR-A, PR-B | ~300 |
| **T2 · Consultar usuarios** | CA-01, CA-02 | PR-C | ~230 |
| **T3 · Modificar usuarios** | CA-03, CA-04, CA-05 | PR-D, PR-E | ~400 |

```
PR-A ──┐
       ├──► PR-C ──► PR-D ──► PR-E
PR-B ──┘
```

PR-A y PR-B son independientes entre sí y pueden ir en paralelo. PR-B **debe** estar
mergeado antes de PR-D: sin R-06, editar el rol de alguien no tiene efecto hasta que su
token expira.

Reglas para todos los PRs (skill `change-size`):

- Cada commit hace **una** cosa y lleva sus pruebas. Si el mensaje necesita un «y», son
  dos commits.
- **Refactor y cambio de comportamiento nunca en el mismo commit.**
- La colección Postman (JSON) no cuenta para el tamaño; se avisa en la descripción del PR.

---

### T1 · PR-A — `fix(common)`: errores de parámetros y paginación (~180)

Hoy `PageRequest` no se usa en ningún sitio, así que se puede cambiar sin romper nada.
Defectos que corrige:

- Sin `sort` revienta: el valor por defecto `"nombre,asc"` no se separa por la coma.
- `CAMPO_ORDEN_INVALIDO` sale con `409` porque usa `ConflictoNegocioException`; el
  contrato pide `400`.
- `size > 100` se recorta en silencio; el contrato pide `400`.
- `sort=nombre` sin dirección ordena descendente.
- `DELETE`, `page=abc` y el JSON malformado caen en `500 ERROR_INTERNO`.

| # | Commit | Tipo | Líneas |
|---|---|---|---|
| A1 | `feat(common): excepción ParametroInvalido con respuesta 400` | comportamiento | ~30 |
| A2 | `fix(common): responder 405 METODO_NO_PERMITIDO en método no soportado` | comportamiento | ~30 |
| A3 | `fix(common): responder 400 PARAMETRO_INVALIDO en parámetros mal formados` | comportamiento | ~40 |
| A4 | `refactor(pagination): lista blanca de orden recibida por recurso` | **refactor** | ~25 |
| A5 | `fix(pagination): sort por defecto, asc implícito y 400 en sort/size` | comportamiento | ~55 |

### T1 · PR-B — `fix(security)`: permisos leídos de la base (R-06) (~120)

| # | Commit | Tipo | Líneas |
|---|---|---|---|
| B1 | `feat(usuario): consulta de usuario con rol en una sola query` | comportamiento | ~15 |
| B2 | `fix(security): construir principal y rol desde la base en cada petición` | comportamiento | ~105 |

B2 se pasa de 50 líneas: unas 25 son el cambio del filtro y el resto la prueba que
demuestra que un token antiguo recibe el rol nuevo. No se separan: una prueba sin el
cambio que verifica no tiene sentido como commit propio.

### T2 · PR-C — `feat(usuario)`: listar y obtener usuarios (CA-01, CA-02) (~230)

| # | Commit | Tipo | Líneas |
|---|---|---|---|
| C1 | `refactor(usuario): extraer la conversión a UsuarioResponse` | **refactor** | ~20 |
| C2 | `feat(usuario): búsqueda paginada por prefijo, rol y estado` | comportamiento | ~70 |
| C3 | `feat(usuario): obtener usuario por id con 404` | comportamiento | ~45 |
| C4 | `feat(usuario): GET /usuarios y GET /usuarios/{id}` | comportamiento | ~95 |

C4 incluye Swagger y la carpeta `HU-02 · Consultar` de la colección Postman.

### T3 · PR-D — `feat(usuario)`: editar usuario (CA-03) (~230)

| # | Commit | Tipo | Líneas |
|---|---|---|---|
| D1 | `feat(usuario): guarda de último administrador con bloqueo` | comportamiento | ~70 |
| D2 | `feat(usuario): editar nombre, correo y rol con unicidad excluyendo al propio` | comportamiento | ~110 |
| D3 | `feat(usuario): PUT /usuarios/{id}` | comportamiento | ~50 |

### T3 · PR-E — `feat(usuario)`: activar y desactivar (CA-04, CA-05) (~170)

| # | Commit | Tipo | Líneas |
|---|---|---|---|
| E1 | `feat(usuario): cambiar estado con auto-desactivación y último admin` | comportamiento | ~90 |
| E2 | `feat(usuario): PATCH /usuarios/{id}/estado` | comportamiento | ~50 |
| E3 | `docs(HU-02): Hechos y manual de consumo del Frontend` | docs | ~30 escritas |

E3 corrige además, en `Docs/Hechos/HU-01-registro-usuarios.md`, la referencia «hasta que
se entregue HU-03» (era HU-02), y deja un aviso de obsoleto en el manual de consumo de
HU-01 (todavía habla del 409 falso, de `estado` en vez de `activo` y de login → 401).

---

## 2. Diseño de la solución

### 2.1 PR-A · Errores y paginación

**A1** — `common/error/ParametroInvalidoException`:

```java
public class ParametroInvalidoException extends ApiException {
    public ParametroInvalidoException(String codigo, String mensaje) {
        super(codigo, HttpStatus.BAD_REQUEST, mensaje);
    }
}
```

`GlobalExceptionHandler` gana un `@ExceptionHandler(ParametroInvalidoException.class)`
igual a los de 404/409. Con eso se pueden lanzar `PARAMETRO_INVALIDO` y
`CAMPO_ORDEN_INVALIDO` como 400.

**A2 / A3** — manejadores nuevos en `GlobalExceptionHandler`, **antes** del genérico
`Exception`:

| Excepción de Spring | HTTP | `code` |
|---|---|---|
| `HttpRequestMethodNotSupportedException` | 405 | `METODO_NO_PERMITIDO` |
| `MethodArgumentTypeMismatchException` (`page=abc`) | 400 | `PARAMETRO_INVALIDO` |
| `MissingServletRequestParameterException` | 400 | `PARAMETRO_INVALIDO` |
| `HttpMessageNotReadableException` (JSON roto) | 400 | `VALIDACION_FALLIDA` |

El mensaje es de dominio (`"El parámetro 'page' no es válido."`), nunca el texto de la
excepción.

**A4 (refactor)** — la lista blanca deja de estar fija en `PageRequest` y la aporta cada
recurso. Mismo comportamiento, firma nueva:

```java
public record PageRequest(int page, int size, String propiedad, boolean ascendente) {
    public static PageRequest of(int page, int size, String sort,
                                 Map<String, String> camposOrden, String porDefecto) { ... }
    public Pageable toPageable() { ... }   // org.springframework.data.domain.PageRequest
}
```

> El nombre choca con `org.springframework.data.domain.PageRequest`. Dentro de
> `toPageable()` se usa el nombre completo; si molesta, renombrarlo
> (`Paginacion`) va en su propio commit de refactor, aprovechando que hoy no tiene usos.

**A5 (comportamiento)** — reglas del contrato §4:

```java
if (page < 0)              throw new ParametroInvalidoException("PARAMETRO_INVALIDO", ...);
if (size < 1 || size > 100) throw new ParametroInvalidoException("PARAMETRO_INVALIDO", ...);
var partes = (sort == null || sort.isBlank() ? porDefecto : sort).split(",");
var campo  = partes[0].trim();
var prop   = camposOrden.get(campo);
if (prop == null) throw new ParametroInvalidoException("CAMPO_ORDEN_INVALIDO", ...);
var asc = partes.length < 2 || !"desc".equalsIgnoreCase(partes[1].trim());
```

Pruebas: sin `sort` → `nombre asc`; `sort=nombre` → asc; `sort=password,asc` → 400;
`size=0`, `size=101`, `page=-1` → 400. `PageResponse.of` no cambia.

### 2.2 PR-B · Principal desde la base (R-06)

**B1** — en `UsuarioRepository`:

```java
@Query("select u from Usuario u join fetch u.rol where u.idUsuario = :id")
Optional<Usuario> findConRolById(@Param("id") Integer id);
```

`rol` es `LAZY` y el filtro corre **fuera** de transacción y de *open-in-view*: acceder a
`getRol()` sobre un `findById` normal lanzaría `LazyInitializationException`.

**B2** — `JwtAuthenticationFilter`:

```java
var usuario = usuarioRepository.findConRolById(Integer.valueOf(claims.getSubject()));
if (usuario.isEmpty() || !Boolean.TRUE.equals(usuario.get().getEstado())) { ...403... }
var u   = usuario.get();
var rol = u.getRol().getNombre();
var principal = new UsuarioAutenticado(u.getIdUsuario(), u.getCorreo(), u.getNombre(), rol);
var authorities = List.of(new SimpleGrantedAuthority("ROLE_" + rol));
```

Los *claims* de `rol`, `correo` y `nombre` se siguen emitiendo (contrato §2, el frontend
los usa para pintar el menú) pero el backend **ya no confía en ellos**.

Pruebas (`JwtAuthenticationFilterTest`): token con claim `ADMIN` de un usuario cuyo rol en
la base es `VENTAS` → la authority resultante es `ROLE_VENTAS`. Se mantienen los casos de
inactivo e inexistente.

### 2.3 PR-C · Listar y obtener

**C1 (refactor)** — `UsuarioService.crear` arma el `UsuarioResponse` a mano. Se extrae a
`private static UsuarioResponse aResponse(Usuario u)` (o `UsuarioResponse.de(u)`) para
reutilizarlo en listar, obtener, editar y cambiar estado. Sin cambio de comportamiento:
pasan los tests de HU-01 tal cual.

**C2** — consulta en `UsuarioRepository`:

```java
@Query(value = """
    select u from Usuario u join fetch u.rol r
    where (:q is null or u.nombre like :q escape '\\' or u.correo like :q escape '\\')
      and (:idRol is null or r.idRol = :idRol)
      and (:activo is null or u.estado = :activo)
    """,
    countQuery = """
    select count(u) from Usuario u join u.rol r
    where (:q is null or u.nombre like :q escape '\\' or u.correo like :q escape '\\')
      and (:idRol is null or r.idRol = :idRol)
      and (:activo is null or u.estado = :activo)
    """)
Page<Usuario> buscar(@Param("q") String q, @Param("idRol") Integer idRol,
                     @Param("activo") Boolean activo, Pageable pageable);
```

El `countQuery` explícito es necesario porque `join fetch` no se puede contar. **Las dos
consultas deben tener el mismo `where`**: es el equivalente JPA de «`buscar()` y
`contar()` comparten el WHERE». Si divergen, `totalElements` miente.

En el servicio:

```java
private static final Map<String, String> ORDEN = Map.of(
    "nombre", "nombre", "correo", "correo", "id", "idUsuario");

static String patronPrefijo(String texto) {          // null/blank → null (sin filtro)
    var t = texto.trim().toLowerCase()
                 .replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_");
    return t + "%";
}
```

`listar(...)` arma `PageRequest.of(page, size, sort, ORDEN, "nombre,asc")`, llama a
`buscar` y devuelve `PageResponse.of(content, page, size, total)`.

**C3** — `obtener(id)`: `findConRolById(id).orElseThrow(() -> new
RecursoNoEncontradoException("el usuario", id))` → `404 RECURSO_NO_ENCONTRADO`.

**C4** — controlador:

```java
@GetMapping
@PreAuthorize("hasRole('ADMIN')")
public PageResponse<UsuarioResponse> listar(
        @RequestParam(required = false) String busqueda,
        @RequestParam(required = false) Integer idRol,
        @RequestParam(required = false) Boolean activo,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(required = false) String sort) { ... }

@GetMapping("/{id}")
@PreAuthorize("hasRole('ADMIN')")
public UsuarioResponse obtener(@PathVariable Integer id) { ... }
```

Se actualiza el `@Tag` del controlador: «Gestión de usuarios del sistema (HU-01, HU-02)».

### 2.4 PR-D · Editar

**D1** — guarda única de R-02, reutilizada por PR-E:

```java
// UsuarioRepository
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select u from Usuario u where u.rol.nombre = 'ADMIN' and u.estado = true")
List<Usuario> bloquearAdminsActivos();

// UsuarioService — se llama dentro de la transacción del cambio
private void validarQuedaAdmin(Usuario objetivo) {
    boolean esAdminActivo = "ADMIN".equals(objetivo.getRol().getNombre())
                            && Boolean.TRUE.equals(objetivo.getEstado());
    if (esAdminActivo && usuarioRepository.bloquearAdminsActivos().size() <= 1) {
        throw new ConflictoNegocioException("ULTIMO_ADMIN",
            "No se puede dejar el sistema sin un administrador activo.");
    }
}
```

El bloqueo (`SELECT ... FOR UPDATE`) serializa dos desactivaciones cruzadas simultáneas:
la segunda espera a la primera y ya ve un solo administrador.

**D2** — DTO y servicio:

```java
public record ActualizarUsuarioRequest(
    @NotBlank @Size(max = 100) String nombre,
    @NotBlank @Email @Size(max = 100) String correo,
    @NotNull Integer idRol) {}
```

`password` y `activo` no existen en el record: Jackson los ignora (Spring Boot desactiva
`FAIL_ON_UNKNOWN_PROPERTIES`). **CP-12 lo verifica**; si Boot 4 / Jackson 3 lo rechazara
con 400, se documenta como desviación y se ajusta CP-12, no se añade el campo.

```java
@Transactional
public UsuarioResponse actualizar(Integer id, ActualizarUsuarioRequest req) {
    var usuario = obtenerEntidad(id);                         // 404
    var correo  = req.correo().trim().toLowerCase();
    var rolNuevo = rolRepository.findById(req.idRol())
        .orElseThrow(() -> new ConflictoNegocioException("ROL_INEXISTENTE", ...));

    boolean cambiaRol = !rolNuevo.getIdRol().equals(usuario.getRol().getIdRol());
    if (cambiaRol && id.equals(SecurityUtils.idUsuarioActual()))
        throw new ConflictoNegocioException("AUTO_DEGRADACION",
            "No puede cambiar su propio rol.");
    if (cambiaRol && !"ADMIN".equals(rolNuevo.getNombre()))
        validarQuedaAdmin(usuario);

    if (usuarioRepository.existsByCorreoAndIdUsuarioNot(correo, id))
        throw new ConflictoNegocioException("CORREO_DUPLICADO", ...);

    usuario.setNombre(req.nombre().trim());
    usuario.setCorreo(correo);
    usuario.setRol(rolNuevo);
    try {
        usuarioRepository.saveAndFlush(usuario);
    } catch (DataIntegrityViolationException e) {            // carrera contra el UNIQUE
        throw new ConflictoNegocioException("CORREO_DUPLICADO", ...);
    }
    return aResponse(usuario);
}
```

> **Por qué `saveAndFlush` y `DataIntegrityViolationException`:** con JPA el `UPDATE` no
> se ejecuta hasta el *flush*. Con `save()` el error saldría al hacer commit, fuera del
> `try`, y llegaría como `409 CONFLICTO_INTEGRIDAD` genérico. Hibernate tampoco lanza
> `DuplicateKeyException` (el `catch` actual de `crear()` en HU-01 tiene el mismo
> problema; se corrige en un commit aparte solo si un test lo demuestra).

Pruebas unitarias: guardar sin cambiar correo → OK (R-04); correo ajeno → 409; rol
inexistente → 409; cambiar el propio rol → `AUTO_DEGRADACION`; degradar al último admin →
`ULTIMO_ADMIN`; degradar a un admin habiendo dos → OK; correo con mayúsculas y espacios →
se guarda normalizado.

**D3** — `@PutMapping("/{id}")` + `@Valid @RequestBody ActualizarUsuarioRequest` +
Swagger (200, 400, 403, 404, 409). Carpeta `HU-02 · Editar` en Postman.

### 2.5 PR-E · Cambiar estado

**E1**:

```java
public record CambiarEstadoRequest(
    @NotNull(message = "El estado es obligatorio") Boolean activo) {}

@Transactional
public UsuarioResponse cambiarEstado(Integer id, CambiarEstadoRequest req) {
    var usuario = obtenerEntidad(id);                         // 404
    if (!req.activo()) {
        if (id.equals(SecurityUtils.idUsuarioActual()))
            throw new ConflictoNegocioException("AUTO_DESACTIVACION",
                "No puede desactivar su propio usuario.");
        validarQuedaAdmin(usuario);
    }
    usuario.setEstado(req.activo());                          // idempotente si ya lo estaba
    return aResponse(usuarioRepository.saveAndFlush(usuario));
}
```

Pruebas: desactivar a otro → OK; a sí mismo → `AUTO_DESACTIVACION`; al último admin →
`ULTIMO_ADMIN`; reactivar → OK sin consultar admins; desactivar a alguien ya inactivo →
200 (idempotente).

**E2** — `@PatchMapping("/{id}/estado")` + Swagger. Carpeta `HU-02 · Estado` en Postman,
con el flujo de CP-14: login de un usuario, desactivarlo con el admin, reusar su token →
`403 USUARIO_INACTIVO`.

CA-05 no requiere código: al no existir `@DeleteMapping`, PR-A ya hace que el `DELETE`
responda `405 METODO_NO_PERMITIDO`.

---

## 3. Archivos

```
Backend/src/main/java/co/edu/corposucre/productionfood/
├── common/error/
│   ├── ParametroInvalidoException.java      (nuevo · A1)
│   └── GlobalExceptionHandler.java          (+400/405 · A1–A3)
├── common/pagination/PageRequest.java       (A4, A5)
├── security/JwtAuthenticationFilter.java    (B2)
└── usuario/
    ├── UsuarioRepository.java               (+findConRolById, buscar,
    │                                          existsByCorreoAndIdUsuarioNot,
    │                                          bloquearAdminsActivos)
    ├── UsuarioService.java                  (+listar, obtener, actualizar, cambiarEstado)
    ├── UsuarioController.java               (+GET, GET/{id}, PUT, PATCH)
    └── dto/{ActualizarUsuarioRequest,CambiarEstadoRequest}.java
```

**Sin migración Flyway**: el esquema y los índices de `V1`–`V3` bastan.

---

## 4. Puntos de cuidado

- **Las dos consultas de `buscar` (`value` y `countQuery`) comparten exactamente el mismo
  `where`.** Es el error más común de la paginación.
- **`sort` nunca llega crudo a la consulta**: se traduce con el `Map` de lista blanca a la
  propiedad JPA (`id → idUsuario`).
- **Escapar `\`, `%` y `_`** en el término de búsqueda y declarar `escape '\\'` en la JPQL.
- **R-02 tiene dos vías** (`PUT` de rol y `PATCH` de estado) y **una sola guarda**
  (`validarQuedaAdmin`) con bloqueo pesimista.
- **El rol se lee de la base en cada petición (R-06).** Sin PR-B, PR-D no está terminado.
- `PUT` no acepta `password` ni `activo` (R-03).
- Nada de `LOWER()` sobre columnas: anula el índice y la colación ya ignora mayúsculas.
- No se escribe en bitácora (R-07).

---

## 5. Checklist de cierre

- [ ] T1, T2 y T3 mergeadas en `main`, cada PR por debajo de ~250 líneas leídas (o con la
      razón del exceso escrita en la descripción).
- [ ] Todos los CA implementados y verificados con la colección Postman HU-02.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize("hasRole('ADMIN')")` en los cuatro métodos nuevos.
- [ ] Pruebas unitarias de cada regla con ramificación (R-02, R-04, R-06, paginación).
- [ ] `mvn test` en verde y newman sin fallos.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] `Docs/Hechos/HU-02-gestion-usuarios.md` y manual de consumo del Frontend escritos (E3).
- [ ] ~~Operaciones auditadas en bitácora~~ → diferido a HU-04 (R-07), anotado en Hechos.
