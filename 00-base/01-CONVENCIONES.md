# 01 — Convenciones del Proyecto

Documento normativo. Cualquier código que no siga estas convenciones se rechaza en revisión.

---

## 1. Idioma

| Elemento | Idioma | Ejemplo |
|---|---|---|
| Tablas y columnas de BD | Español (tal como está en `db.sql`) | `materias_primas.stock_minimo` |
| Rutas de API | Español | `/api/v1/materias-primas` |
| Clases, métodos, variables Java | Español para el dominio, inglés para lo técnico | `MateriaPrimaService`, `findById` |
| Componentes y servicios Angular | Español para el dominio | `materia-prima-list.component.ts` |
| Mensajes al usuario final | Español | `"El correo ya está registrado"` |
| Comentarios y documentación | Español | — |
| Códigos de error internos | Inglés, SCREAMING_SNAKE | `EMAIL_ALREADY_EXISTS` |

**Regla anti-ambigüedad:** nunca mezclar `materiaPrima` y `rawMaterial` en el mismo flujo.
El dominio es español, punto.

---

## 2. Backend — estructura de paquetes

Paquete raíz: `co.edu.corposucre.productionfood`

```
co.edu.corposucre.productionfood
├── ProductionFoodApplication.java
├── config/
│   ├── OpenApiConfig.java           Configuración de Swagger/SpringDoc
│   ├── DataSourceConfig.java        JdbcTemplate + NamedParameterJdbcTemplate
│   └── WebConfig.java               CORS
├── security/
│   ├── SecurityConfig.java          SecurityFilterChain, reglas por ruta
│   ├── JwtService.java              Firma y validación de tokens
│   ├── JwtAuthenticationFilter.java Filtro OncePerRequest
│   ├── UsuarioDetailsService.java   Carga de usuario para autenticación
│   └── SecurityUtils.java           Acceso al usuario autenticado actual
├── common/
│   ├── error/
│   │   ├── ApiException.java        Excepción base con código y estado HTTP
│   │   ├── RecursoNoEncontradoException.java
│   │   ├── ConflictoNegocioException.java
│   │   ├── ErrorResponse.java       DTO de error (ver 04-CONTRATO-API.md)
│   │   └── GlobalExceptionHandler.java  @RestControllerAdvice
│   ├── pagination/
│   │   ├── PageRequest.java         page, size, sort — validado
│   │   └── PageResponse.java        content, page, size, totalElements, totalPages
│   └── audit/
│       ├── Auditable.java           Anotación de método
│       ├── AuditoriaAspect.java     AOP que escribe en bitacora
│       └── BitacoraService.java
└── <modulo>/                         Un paquete por agregado de negocio
    ├── UsuarioController.java
    ├── UsuarioService.java
    ├── UsuarioRepository.java
    ├── UsuarioRowMapper.java
    ├── Usuario.java                  Entidad (POJO plano, sin anotaciones JPA)
    └── dto/
        ├── CrearUsuarioRequest.java
        ├── ActualizarUsuarioRequest.java
        └── UsuarioResponse.java
```

### Módulos previstos

`usuario`, `rol`, `auth`, `bitacora`, `cliente`, `proveedor`, `unidadmedida`,
`materiaprima`, `lote`, `inventario`, `producto`, `receta`, `compra`, `pedido`,
`devolucion`, `produccion`, `reporte`.

### Responsabilidad de cada capa

| Capa | Sí hace | No hace |
|---|---|---|
| `Controller` | Mapear HTTP ↔ DTO, validar con `@Valid`, declarar `@PreAuthorize` | Lógica de negocio, SQL, transacciones |
| `Service` | Reglas de negocio, orquestación, `@Transactional`, auditoría | Construir SQL, conocer `HttpServletRequest` |
| `Repository` | SQL con `JdbcTemplate`, `RowMapper`, paginación | Reglas de negocio, lanzar excepciones de dominio |
| `RowMapper` | `ResultSet` → entidad | Cualquier otra cosa |

**Sin ORM.** Todo el SQL es explícito y vive en el repositorio. Nunca en el servicio,
nunca en el controlador.

---

## 3. Backend — reglas de código

### SQL

- **Siempre parametrizado.** Cero concatenación de valores. Ni siquiera "porque es un int".
  ```java
  // MAL
  jdbc.query("SELECT * FROM usuarios WHERE id_usuario = " + id, mapper);
  // BIEN
  jdbc.query("SELECT * FROM usuarios WHERE id_usuario = ?", mapper, id);
  ```
- **Filtros dinámicos** con `NamedParameterJdbcTemplate` y `StringBuilder` sobre el `WHERE`,
  nunca interpolando el valor:
  ```java
  var sql = new StringBuilder("SELECT ... FROM clientes WHERE 1=1");
  var params = new MapSqlParameterSource();
  if (nombre != null) { sql.append(" AND nombre LIKE :nombre"); params.addValue("nombre", "%" + nombre + "%"); }
  ```
- **`ORDER BY` dinámico**: el campo de ordenamiento se valida contra una lista blanca
  (`Set.of("nombre","correo","id_usuario")`) antes de concatenarse. Nunca se acepta el
  valor crudo del cliente.
- **Nunca `SELECT *`** en producción: se listan las columnas, para que un `ALTER TABLE`
  no rompa el `RowMapper` en silencio.

### Transacciones

- `@Transactional` va en el **servicio**, nunca en el repositorio ni en el controlador.
- Toda operación que toque más de una tabla es transaccional. Sin excepciones.
- `@Transactional(readOnly = true)` en consultas: es documentación ejecutable.

### Validación

- Bean Validation (`jakarta.validation`) en los DTO de request para lo sintáctico:
  obligatoriedad, longitud, rango, formato.
- Reglas que requieren consultar la base (unicidad, existencia de FK, stock suficiente)
  van en el **servicio**, no en anotaciones.

### Nulos

- Los repositorios devuelven `Optional<T>` para búsquedas por id.
- El servicio convierte `Optional.empty()` en `RecursoNoEncontradoException`.
- Los controladores nunca ven un `null` de dominio.

---

## 4. Frontend — estructura

Angular 17+ con **componentes standalone** (sin `NgModule` por feature).

```
src/app/
├── app.config.ts                 Providers raíz, interceptores, router
├── app.routes.ts                 Rutas de primer nivel con lazy loading
├── core/
│   ├── auth/
│   │   ├── auth.service.ts       login, logout, token, usuario actual
│   │   ├── auth.guard.ts         canActivate — sesión válida
│   │   ├── rol.guard.ts          canActivate — rol autorizado
│   │   └── auth.interceptor.ts   Adjunta Authorization: Bearer
│   ├── http/
│   │   ├── error.interceptor.ts  401→logout, 403→aviso, 5xx→snackbar
│   │   └── api.config.ts         baseUrl desde environment
│   └── ui/
│       ├── notificacion.service.ts  Wrapper de MatSnackBar
│       └── confirmacion.service.ts  Wrapper de MatDialog para confirmar
├── shared/
│   ├── components/
│   │   ├── tabla-paginada/       Tabla genérica con MatPaginator + MatSort
│   │   ├── barra-filtros/
│   │   ├── estado-chip/          Chip de color según estado
│   │   └── confirmar-dialog/
│   ├── pipes/                    moneda, cantidad, estado-activo
│   └── models/                   Interfaces compartidas (PageResponse, ApiError)
└── features/
    └── <modulo>/
        ├── <modulo>.routes.ts
        ├── services/<modulo>.service.ts
        ├── models/<modulo>.model.ts
        └── pages/
            ├── <modulo>-lista/
            └── <modulo>-formulario/
```

### Reglas frontend

- **Reactive Forms siempre.** Nada de `ngModel` en formularios de dominio.
- **Un servicio HTTP por módulo**, tipado con interfaces. Nunca `any` en la respuesta.
- **Los componentes no llaman a `HttpClient` directamente**, solo a su servicio.
- **Estado local por componente** con signals. No se introduce NgRx en este alcance.
- **Modelos TypeScript espejo de los DTO del backend.** Si cambia el DTO, cambia la
  interfaz en el mismo PR.
- **Sin lógica de negocio duplicada:** la validación del cliente es para experiencia de
  usuario; la autoridad es el backend. Que el formulario deshabilite el botón no exime
  al servidor de validar.

---

## 5. Nomenclatura

### Rutas de API

- Plural, kebab-case: `/api/v1/materias-primas`, `/api/v1/planes-produccion`
- Subrecursos anidados cuando el hijo no existe sin el padre:
  `/api/v1/compras/{idCompra}/detalles`, `/api/v1/recetas/{idReceta}/detalles`
- Acciones que no son CRUD, como subrecurso de estado:
  `PATCH /api/v1/pedidos/{id}/estado`, `POST /api/v1/compras/{id}/recepcion`

### Java

| Elemento | Convención | Ejemplo |
|---|---|---|
| Clase | PascalCase | `MateriaPrimaService` |
| Método | camelCase, verbo primero | `buscarPorNombre`, `registrarConsumo` |
| Constante | SCREAMING_SNAKE | `MAX_PAGE_SIZE` |
| DTO de entrada | `<Accion><Entidad>Request` | `CrearClienteRequest` |
| DTO de salida | `<Entidad>Response` | `ClienteResponse` |

### Angular

| Elemento | Convención | Ejemplo |
|---|---|---|
| Archivo | kebab-case | `materia-prima-lista.component.ts` |
| Clase | PascalCase + sufijo | `MateriaPrimaListaComponent` |
| Interface | PascalCase, sin prefijo `I` | `MateriaPrima`, `PageResponse<T>` |
| Observable | sufijo `$` | `materiasPrimas$` |

### Base de datos

Se respeta lo existente en `db.sql`: `snake_case`, tablas en plural, PK `id_<singular>`.
Las tablas nuevas siguen la misma convención.

---

## 6. Git

- **Rama por HU**: `feat/HU-08-materias-primas`, `fix/HU-12-receta-duplicada`
- **Commits atómicos**, bajo ~50 líneas, con el prefijo de la HU:
  ```
  feat(HU-08): agregar MateriaPrimaRepository con filtro por nombre

  Implementa búsqueda paginada con lista blanca de ordenamiento.
  ```
- **Un PR por capa de HU** (backend / frontend), bajo ~250 líneas.
  Una HU grande se parte en varios PR; no se acumula.
- Nunca se hace push directo a `main`.

---

## 7. Definición de Terminado (DoD)

Una HU está terminada cuando **todas** se cumplen:

- [ ] Todos los criterios de aceptación de la HU están implementados y verificados.
- [ ] Los endpoints están documentados en Swagger con ejemplos de request y response.
- [ ] Los casos de prueba de `tarea-qa.md` pasan, incluidos los negativos.
- [ ] El frontend maneja los tres estados: cargando, vacío y error.
- [ ] Las acciones de escritura quedan registradas en bitácora (ver HU-04).
- [ ] La autorización por rol está aplicada y probada con un rol no autorizado.
- [ ] No hay `System.out.println`, `console.log`, ni código comentado.
- [ ] El PR fue revisado y aprobado por alguien distinto de quien lo escribió.
