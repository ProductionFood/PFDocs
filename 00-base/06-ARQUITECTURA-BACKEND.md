# 06 — Arquitectura Backend

Spring Boot 4.1 · Java 25 · JdbcTemplate · MySQL 8.4

> Este documento contiene la configuración **escrita para Spring Security 7**. La mayoría
> de tutoriales de JWT con Spring que se encuentran en línea son para Spring Boot 3.x y no
> compilan aquí. **Si un ejemplo de internet contradice este documento, el ejemplo está
> desactualizado.**

---

## 1. Capas

```
HTTP
 │
 ▼
Controller ──► valida el DTO (@Valid), autoriza (@PreAuthorize), mapea HTTP
 │
 ▼
Service ────► reglas de negocio, @Transactional, auditoría, orquestación
 │
 ▼
Repository ─► SQL con JdbcTemplate, RowMapper, paginación
 │
 ▼
MySQL
```

Sin ORM: todo el SQL es explícito. Las responsabilidades de cada capa están en
`01-CONVENCIONES.md` §2.

---

## 2. Seguridad

### `SecurityConfig` (Spring Security 7)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity          // ⚠️ SIN ESTO, TODAS LAS @PreAuthorize SE IGNORAN
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;
    private final AuthenticationEntryPoint entryPoint;
    private final AccessDeniedHandler accessDeniedHandler;

    public SecurityConfig(JwtAuthenticationFilter jwtFilter,
                          AuthenticationEntryPoint entryPoint,
                          AccessDeniedHandler accessDeniedHandler) {
        this.jwtFilter = jwtFilter;
        this.entryPoint = entryPoint;
        this.accessDeniedHandler = accessDeniedHandler;
    }

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            // API stateless con JWT: CSRF no aplica (no hay cookies de sesión)
            .csrf(AbstractHttpConfigurer::disable)
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/login").permitAll()
                .requestMatchers("/swagger-ui/**", "/swagger-ui.html", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated())
            .exceptionHandling(e -> e
                .authenticationEntryPoint(entryPoint)        // 401 en formato del contrato
                .accessDeniedHandler(accessDeniedHandler))   // 403 en formato del contrato
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();   // HU-01 CA-03
    }

    @Bean
    AuthenticationManager authenticationManager(AuthenticationConfiguration cfg) throws Exception {
        return cfg.getAuthenticationManager();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        var c = new CorsConfiguration();
        c.setAllowedOrigins(List.of("http://localhost:4200"));
        c.setAllowedMethods(List.of("GET","POST","PUT","PATCH","DELETE","OPTIONS"));
        c.setAllowedHeaders(List.of("Authorization","Content-Type"));
        var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", c);
        return source;
    }
}
```

**`@EnableMethodSecurity` es la línea más importante del archivo.** Sin ella la aplicación
arranca igual, los endpoints responden igual, y las `@PreAuthorize` no se evalúan: la
autorización del sistema completo queda desactivada sin un solo mensaje de error. El caso
SEC-04 de `05-ESTANDARES-QA.md` existe para detectar exactamente esto.

**Swagger se cierra fuera de desarrollo** con `@Profile("dev")` sobre una configuración
alternativa, o retirando esas rutas de `permitAll` en el perfil de producción.

### `JwtService`

```java
@Service
public class JwtService {

    private final SecretKey key;
    private final long expiracionMs;

    public JwtService(@Value("${app.jwt.secret}") String secret,
                      @Value("${app.jwt.expiration-ms}") long expiracionMs) {
        // La clave debe tener al menos 256 bits para HS256; jjwt lo verifica y falla si no
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.expiracionMs = expiracionMs;
    }

    public String generar(Usuario u) {                       // HU-03 CA-05
        var ahora = new Date();
        return Jwts.builder()
                .subject(String.valueOf(u.getIdUsuario()))
                .claim("correo", u.getCorreo())
                .claim("nombre", u.getNombre())
                .claim("rol", u.getNombreRol())
                .issuedAt(ahora)
                .expiration(new Date(ahora.getTime() + expiracionMs))
                .signWith(key)
                .compact();
    }

    public Claims validarYExtraer(String token) {
        return Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

Nota sobre jjwt 0.13: la API fluida cambió respecto a 0.11 (`setSubject` → `subject`,
`parserBuilder()` → `parser()`, `signWith(key, alg)` → `signWith(key)`). Los ejemplos
antiguos no compilan.

### Configuración

```properties
app.jwt.secret=${JWT_SECRET}
app.jwt.expiration-ms=3600000
```

> 🔒 El secreto **nunca** se versiona. Un `JWT_SECRET` en el repositorio permite a
> cualquiera firmar tokens válidos con el rol que quiera: equivale a publicar la
> contraseña de administrador. Se genera con `openssl rand -base64 48`.

### `JwtAuthenticationFilter`

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String header = req.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            try {
                Claims claims = jwtService.validarYExtraer(header.substring(7));
                var authorities = List.of(new SimpleGrantedAuthority("ROLE_" + claims.get("rol")));
                var auth = new UsernamePasswordAuthenticationToken(
                        new UsuarioAutenticado(
                                Long.valueOf(claims.getSubject()),
                                claims.get("correo", String.class),
                                claims.get("rol", String.class)),
                        null, authorities);
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException ex) {
                // Token inválido o expirado: se deja sin autenticar.
                // El EntryPoint devuelve 401 NO_AUTENTICADO con el formato del contrato.
                SecurityContextHolder.clearContext();
            }
        }
        chain.doFilter(req, res);
    }
}
```

El prefijo `ROLE_` se agrega aquí porque `hasRole('ADMIN')` busca la authority
`ROLE_ADMIN`. Ver `03-MATRIZ-ROLES.md` §4.

---

## 3. Manejo de errores

Un único `@RestControllerAdvice` produce el formato de `04-CONTRATO-API.md` §5:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ErrorResponse> validacion(MethodArgumentNotValidException ex, HttpServletRequest req) {
        var campos = ex.getBindingResult().getFieldErrors().stream()
                .map(f -> new FieldError(f.getField(), f.getDefaultMessage()))
                .toList();
        return build(HttpStatus.BAD_REQUEST, "VALIDACION_FALLIDA",
                     "Los datos enviados no son válidos.", req, campos);
    }

    @ExceptionHandler(RecursoNoEncontradoException.class)
    ResponseEntity<ErrorResponse> noEncontrado(...) { /* 404 */ }

    @ExceptionHandler(ConflictoNegocioException.class)
    ResponseEntity<ErrorResponse> conflicto(...) { /* 409 con el code de la excepción */ }

    @ExceptionHandler(DataIntegrityViolationException.class)
    ResponseEntity<ErrorResponse> integridad(DataIntegrityViolationException ex, HttpServletRequest req) {
        // Traduce la violación de constraint a un mensaje de dominio.
        // NUNCA se devuelve ex.getMessage(): expone nombres de tabla, columnas y SQL.
        log.warn("Violación de integridad en {}", req.getRequestURI(), ex);
        return build(HttpStatus.CONFLICT, "CONFLICTO_INTEGRIDAD",
                     "La operación entra en conflicto con datos existentes.", req, List.of());
    }

    @ExceptionHandler(Exception.class)
    ResponseEntity<ErrorResponse> general(Exception ex, HttpServletRequest req) {
        log.error("Error no controlado en {}", req.getRequestURI(), ex);
        return build(HttpStatus.INTERNAL_SERVER_ERROR, "ERROR_INTERNO",
                     "Ocurrió un error inesperado. Intente nuevamente.", req, List.of());
    }
}
```

**El detalle técnico va al log, el mensaje de dominio a la respuesta.** Devolver el
`getMessage()` de una excepción de base de datos filtra la estructura interna del sistema
a cualquiera que provoque un error.

---

## 4. Repositorios con JdbcTemplate

### Patrón base

```java
@Repository
public class ClienteRepository {

    private static final String COLUMNAS =
        "id_cliente, nombre, contacto, telefono, estado";

    // Lista blanca de ordenamiento: el valor del cliente nunca llega crudo al SQL
    private static final Map<String,String> ORDEN = Map.of(
        "nombre", "nombre",
        "id",     "id_cliente");

    private final NamedParameterJdbcTemplate jdbc;
    private final ClienteRowMapper mapper;

    public Optional<Cliente> buscarPorId(Long id) {
        var sql = "SELECT " + COLUMNAS + " FROM clientes WHERE id_cliente = :id";
        return jdbc.query(sql, Map.of("id", id), mapper).stream().findFirst();
    }

    public List<Cliente> buscar(String nombre, Boolean activo, PageRequest page) {
        var sql = new StringBuilder("SELECT " + COLUMNAS + " FROM clientes WHERE 1=1");
        var params = new MapSqlParameterSource();

        if (nombre != null && !nombre.isBlank()) {
            sql.append(" AND nombre LIKE :nombre");
            params.addValue("nombre", escaparLike(nombre) + "%");   // prefijo, usa el índice
        }
        if (activo != null) {
            sql.append(" AND estado = :estado");
            params.addValue("estado", activo ? 1 : 0);
        }

        sql.append(" ORDER BY ").append(ORDEN.get(page.campo()))   // ya validado
           .append(page.ascendente() ? " ASC" : " DESC")
           .append(" LIMIT :limit OFFSET :offset");
        params.addValue("limit", page.size());
        params.addValue("offset", page.offset());

        return jdbc.query(sql.toString(), params, mapper);
    }

    /** Escapa los comodines de LIKE. Sin esto, buscar "%" devuelve todos los registros. */
    private String escaparLike(String v) {
        return v.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_");
    }
}
```

### Insertar y recuperar el id generado

```java
public Long insertar(Cliente c) {
    var sql = "INSERT INTO clientes (nombre, contacto, telefono, estado) " +
              "VALUES (:nombre, :contacto, :telefono, :estado)";
    var params = new MapSqlParameterSource()
        .addValue("nombre", c.getNombre())
        .addValue("contacto", c.getContacto())
        .addValue("telefono", c.getTelefono())
        .addValue("estado", c.isActivo() ? 1 : 0);

    var keyHolder = new GeneratedKeyHolder();
    jdbc.update(sql, params, keyHolder, new String[]{"id_cliente"});
    return keyHolder.getKey().longValue();
}
```

### RowMapper

```java
@Component
public class ClienteRowMapper implements RowMapper<Cliente> {
    @Override
    public Cliente mapRow(ResultSet rs, int rowNum) throws SQLException {
        var c = new Cliente();
        c.setIdCliente(rs.getLong("id_cliente"));
        c.setNombre(rs.getString("nombre"));
        c.setContacto(rs.getString("contacto"));
        c.setTelefono(rs.getString("telefono"));
        c.setActivo(rs.getBoolean("estado"));   // tinyint(1) → boolean, según el contrato
        return c;
    }
}
```

### Conteo para la paginación

```java
public long contar(String nombre, Boolean activo) {
    // Mismo WHERE que buscar(). Si divergen, totalElements miente.
    ...
}
```

Mantener sincronizados el `WHERE` de `buscar()` y el de `contar()` es la fuente de error más
frecuente en paginación manual: la lista filtra y el contador no, y la última página aparece
vacía.

---

## 5. Transacciones y bloqueo de inventario

Toda operación que toca más de una tabla es transaccional:

```java
@Service
public class ConsumoService {

    @Transactional                                   // HU-23 CA-04
    public ConsumoResponse registrar(Long idDetallePlan, List<ConsumoRequest> consumos) {
        for (var c : consumos) {
            // SELECT ... FOR UPDATE: bloquea la fila hasta el fin de la transacción.
            // Sin esto, dos producciones simultáneas leen el mismo saldo, ambas validan
            // y el inventario queda negativo.
            var saldo = inventarioRepository.bloquearYObtener(c.idMateriaPrima());

            if (saldo.compareTo(c.cantidadReal()) < 0) {
                throw new ConflictoNegocioException("STOCK_INSUFICIENTE", "...");
            }
            inventarioService.registrarSalida(
                c.idMateriaPrima(), c.cantidadReal(),
                TipoMovimiento.SALIDA_PRODUCCION, "consumo_materia_prima", idConsumo);
        }
        return ...;
    }
}
```

```java
// InventarioRepository
public BigDecimal bloquearYObtener(Long idMateriaPrima) {
    var sql = "SELECT cantidad_disponible FROM inventario " +
              "WHERE id_materia_prima = :id FOR UPDATE";
    ...
}
```

**Regla del inventario:** `InventarioService` es el **único** componente que ejecuta
`UPDATE inventario`, y siempre inserta el movimiento de kardex en la misma transacción
(`02-CORRECCIONES-DB.md` §C-04). Ningún otro servicio toca esas tablas directamente.

---

## 6. Auditoría (HU-04)

Aspecto AOP sobre los métodos anotados:

```java
@Auditable(accion = "CREAR", tabla = "clientes")
public ClienteResponse crear(CrearClienteRequest req) { ... }
```

```java
@Aspect
@Component
public class AuditoriaAspect {

    @AfterReturning(pointcut = "@annotation(auditable)", returning = "resultado")
    public void registrar(JoinPoint jp, Auditable auditable, Object resultado) {
        bitacoraService.registrar(
            SecurityUtils.idUsuarioActual(),   // null si es acción del sistema (§C-11)
            auditable.accion(),
            auditable.tabla(),
            resumen(resultado));               // recortado a 255 caracteres
    }
}
```

`@AfterReturning`, no `@Before`: solo se audita lo que efectivamente ocurrió. Auditar antes
registra intentos fallidos como si fueran cambios reales.

**El fallo al escribir en bitácora no debe tumbar la operación de negocio.**
`BitacoraService.registrar` captura sus propias excepciones y las registra en el log. Que la
auditoría falle es un problema; que impida vender, es peor.

---

## 7. Estructura de un endpoint completo

```java
@RestController
@RequestMapping("/api/v1/clientes")
@Tag(name = "Clientes", description = "Gestión de clientes (HU-05)")
public class ClienteController {

    @GetMapping
    @PreAuthorize("hasAnyRole('ADMIN','VENTAS','PRODUCCION','CONSULTA')")
    @Operation(summary = "Listar clientes con filtro y paginación")
    public PageResponse<ClienteResponse> listar(
            @RequestParam(required = false) String nombre,
            @RequestParam(required = false) Boolean activo,
            @RequestParam(defaultValue = "0")  @Min(0) int page,
            @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size,
            @RequestParam(defaultValue = "nombre,asc") String sort) {
        return clienteService.listar(nombre, activo, PageRequest.of(page, size, sort));
    }

    @PostMapping
    @PreAuthorize("hasAnyRole('ADMIN','VENTAS')")
    @Operation(summary = "Registrar un cliente")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Cliente creado"),
        @ApiResponse(responseCode = "400", description = "Datos inválidos"),
        @ApiResponse(responseCode = "403", description = "Sin permisos")
    })
    public ResponseEntity<ClienteResponse> crear(@Valid @RequestBody CrearClienteRequest req) {
        var creado = clienteService.crear(req);
        return ResponseEntity
                .created(URI.create("/api/v1/clientes/" + creado.idCliente()))
                .body(creado);
    }
}
```

`@Max(100)` en `size` implementa el tope de `04-CONTRATO-API.md` §4.

---

## 8. DTO con records

Java 25; los DTO son `record`:

```java
public record CrearClienteRequest(
    @NotBlank(message = "El nombre es obligatorio")
    @Size(max = 100, message = "El nombre no puede superar 100 caracteres")
    String nombre,

    @Size(max = 100) String contacto,
    @Size(max = 20)  String telefono
) {}

public record ClienteResponse(
    Long idCliente, String nombre, String contacto, String telefono, boolean activo
) {}
```

Los mensajes de validación van **en español**: llegan al usuario tal cual a través de
`fieldErrors`.

---

## 9. Pruebas

```java
@Test
void noPermiteConsumirMasStockDelDisponible() {
    when(inventarioRepository.bloquearYObtener(3L)).thenReturn(new BigDecimal("10.00"));

    var ex = assertThrows(ConflictoNegocioException.class,
        () -> consumoService.registrar(1L, List.of(new ConsumoRequest(3L, ..., new BigDecimal("15.00")))));

    assertEquals("STOCK_INSUFICIENTE", ex.getCodigo());
    verify(inventarioService, never()).registrarSalida(any(), any(), any(), any(), any());
}
```

Lo que se prueba obligatoriamente (`05-ESTANDARES-QA.md` §1): cálculos, validaciones de
stock, transiciones de estado y la fórmula de eficiencia con `cantidad_real = 0`.
